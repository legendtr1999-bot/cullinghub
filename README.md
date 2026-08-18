--[[
SosyHUB — Cascade Feature Merge
Source 1: SosyHUB_Bastion_UI_FIXED16_Short_Purple_Search(1).lua
Source 2: d7edc003-0298-40d7-8894-107891772dd3.lua

This build keeps the Cascade/macOS-style UI from source 2 and mounts the
feature inventory, backend callbacks, sliders, toggles, lists, buttons and
native feature engine from source 1.
]]

local function runEmbed(src, label)
    local fn, err = loadstring(src, label)
    if not fn then error("[SosyHUB] " .. tostring(label) .. " compile: " .. tostring(err)) end
    local ok, result = pcall(fn)
    if not ok then error("[SosyHUB] " .. tostring(label) .. " runtime: " .. tostring(result)) end
    return result
end

-- ===================== Executor compatibility =====================
-- Solara, Xeno and Delta do not expose the same surface as Potassium/Synapse. Two
-- differences break a script this size:
--   1. Some of them keep request / file functions only inside getgenv(), so the bare
--      globals this script reads resolve to nil.
--   2. Several ship no file API at all, and the config/piano paths call isfolder and
--      writefile directly.
-- Rather than guard 131 call sites, resolve each name once and backfill it into the
-- global table so existing code keeps working unchanged.
do
    local genv = (type(getgenv) == "function") and getgenv() or nil
    local G = genv or _G

    local function resolve(...)
        for _, name in ipairs({ ... }) do
            local v = rawget(G, name)
            if type(v) ~= "function" and genv then v = rawget(genv, name) end
            if type(v) ~= "function" then
                local ok, got = pcall(function() return (loadstring or load)("return " .. name)() end)
                if ok and type(got) == "function" then v = got end
            end
            if type(v) == "function" then return v end
        end
        return nil
    end

    local exec = "unknown"
    pcall(function()
        local f = resolve("identifyexecutor", "getexecutorname")
        if f then exec = tostring(select(1, f()) or "unknown") end
    end)

    local http = resolve("http_request", "request")
    if not http then
        pcall(function() if syn and type(syn.request) == "function" then http = syn.request end end)
    end
    if not http then
        pcall(function() if http and type(http.request) == "function" then http = http.request end end)
    end

    -- In-memory stand-ins so a missing file API degrades to "nothing saved" instead of
    -- erroring out of the whole script on load.
    -- Single-file distribution: never read/write the executor's real filesystem.
    -- Config/auth/theme data is session-only; no local files are required.
    local vfs = {}
    local vdir = {}
    local fileApi = {
        isfolder   = function(p) return vdir[tostring(p)] == true end,
        makefolder = function(p) vdir[tostring(p)] = true end,
        isfile     = function(p) return vfs[tostring(p)] ~= nil end,
        readfile   = function(p)
            local v = vfs[tostring(p)]
            if v == nil then error("file not found: " .. tostring(p), 0) end
            return v
        end,
        writefile  = function(p, c) vfs[tostring(p)] = tostring(c) end,
        appendfile = function(p, c) vfs[tostring(p)] = (vfs[tostring(p)] or "") .. tostring(c) end,
        delfile    = function(p) vfs[tostring(p)] = nil end,
        listfiles  = function() return {} end,
    }
    for name, fn in pairs(fileApi) do
        if type(rawget(G, name)) ~= "function" then G[name] = fn end
    end

    if http and type(rawget(G, "request")) ~= "function" then G.request = http end
    if http and type(rawget(G, "http_request")) ~= "function" then G.http_request = http end

    -- Optional hooks. The features that need these already check for them, but pin the
    -- names into the global table so those checks see what the executor actually has.
    for _, name in ipairs({
        "hookmetamethod", "getnamecallmethod", "newcclosure", "getrawmetatable",
        "setreadonly", "checkcaller", "gethui", "setclipboard", "getcustomasset",
        "queue_on_teleport", "getconnections", "firetouchinterest",
    }) do
        local fn = resolve(name)
        if fn and type(rawget(G, name)) ~= "function" then G[name] = fn end
    end

    local missing = {}
    for _, name in ipairs({ "hookmetamethod", "getnamecallmethod", "newcclosure" }) do
        if type(resolve(name)) ~= "function" then missing[#missing + 1] = name end
    end

    _G.SosyExec = {
        Name = exec,
        HasHttp = http ~= nil,
        HasFiles = resolve("writefile") ~= nil,
        HasHooks = #missing == 0,
        MissingHooks = missing,
    }

    -- Run this on a failing executor and send me the output; it names the gap instead
    -- of leaving us guessing which of the three is short.
    _G.SosyExecReport = function()
        local names = {
            "loadstring", "getgenv", "request", "http_request", "hookmetamethod",
            "getnamecallmethod", "newcclosure", "getrawmetatable", "setreadonly",
            "isfolder", "makefolder", "isfile", "readfile", "writefile", "listfiles",
            "gethui", "setclipboard", "getcustomasset", "queue_on_teleport",
            "getconnections", "firetouchinterest", "checkcaller",
        }
        local have, lack = {}, {}
        for _, n in ipairs(names) do
            if type(resolve(n)) == "function" then have[#have + 1] = n else lack[#lack + 1] = n end
        end
        warn("[SosyHUB] executor = " .. exec)
        warn("[SosyHUB] VAR    : " .. table.concat(have, ", "))
        warn("[SosyHUB] YOK    : " .. table.concat(lack, ", "))
        warn("[SosyHUB] HttpGet: " .. tostring(pcall(function() return game:HttpGet("https://example.com") end)))
        return exec, have, lack
    end

    if #missing > 0 then
        warn("[SosyHUB] " .. exec .. " eksik: " .. table.concat(missing, ", ")
            .. " — Fast Swap gibi hook gerektiren ozellikler kapali kalacak, hub yine acilir.")
    end
end
-- ===================== /Executor compatibility =====================

-- ===================== Theme =====================
local Theme = {
	Surface = Color3.fromRGB(8, 8, 8),
	Elevated = Color3.fromRGB(14, 14, 14),
	Header = Color3.fromRGB(10, 10, 10),
	Hover = Color3.fromRGB(22, 22, 22),
	Accent = Color3.fromRGB(210, 20, 20),
	Border = Color3.fromRGB(38, 38, 38),
	BorderStrong = Color3.fromRGB(80, 20, 20),
	Text = Color3.fromRGB(236, 240, 246),
	TextSecondary = Color3.fromRGB(168, 178, 194),
	TextMuted = Color3.fromRGB(110, 120, 138),
	Success = Color3.fromRGB(74, 222, 128),
	Error = Color3.fromRGB(248, 113, 113),
	ToggleOff = Color3.fromRGB(40, 40, 40),
}

local ThemePresets = {
	-- Accents only — Light/Dark surfaces come from ThemeMode (White Mode toggle)
	Midnight    = { Accent = Color3.fromRGB(70, 140, 255),  Surface = Color3.fromRGB(4, 8, 16),    Elevated = Color3.fromRGB(10, 16, 28) },
	Gold        = { Accent = Color3.fromRGB(255, 196, 40),  Surface = Color3.fromRGB(6, 6, 8),     Elevated = Color3.fromRGB(14, 14, 16) },
	Stealth     = { Accent = Color3.fromRGB(120, 120, 128), Surface = Color3.fromRGB(6, 6, 8),     Elevated = Color3.fromRGB(16, 16, 18) },
	["Soft Pastel"] = { Accent = Color3.fromRGB(255, 100, 150), Surface = Color3.fromRGB(28, 14, 22), Elevated = Color3.fromRGB(40, 22, 32) },
	["Red/Black"] = { Accent = Color3.fromRGB(180, 20, 20), Surface = Color3.fromRGB(15, 15, 15), Elevated = Color3.fromRGB(20, 20, 20) },
	-- ── New themes ──
	Emerald     = { Accent = Color3.fromRGB(52, 211, 153),  Surface = Color3.fromRGB(4, 14, 10),   Elevated = Color3.fromRGB(8, 22, 16) },
	Crimson     = { Accent = Color3.fromRGB(220, 38, 38),   Surface = Color3.fromRGB(14, 4, 4),    Elevated = Color3.fromRGB(22, 8, 8) },
	Ocean       = { Accent = Color3.fromRGB(14, 165, 233),  Surface = Color3.fromRGB(2, 10, 18),   Elevated = Color3.fromRGB(4, 16, 28) },
	Purple      = { Accent = Color3.fromRGB(168, 85, 247),  Surface = Color3.fromRGB(12, 4, 20),   Elevated = Color3.fromRGB(18, 8, 30) },
	Sakura      = { Accent = Color3.fromRGB(244, 114, 182), Surface = Color3.fromRGB(22, 8, 16),   Elevated = Color3.fromRGB(32, 12, 22) },
	Cyber       = { Accent = Color3.fromRGB(0, 255, 204),   Surface = Color3.fromRGB(2, 8, 12),    Elevated = Color3.fromRGB(4, 14, 18) },
	Obsidian    = { Accent = Color3.fromRGB(148, 163, 184), Surface = Color3.fromRGB(4, 4, 6),     Elevated = Color3.fromRGB(10, 10, 14) },
	Amber       = { Accent = Color3.fromRGB(251, 146, 60),  Surface = Color3.fromRGB(14, 6, 2),    Elevated = Color3.fromRGB(22, 10, 4) },
	Lime        = { Accent = Color3.fromRGB(163, 230, 53),  Surface = Color3.fromRGB(6, 12, 2),    Elevated = Color3.fromRGB(10, 18, 4) },
	Rose        = { Accent = Color3.fromRGB(251, 113, 133), Surface = Color3.fromRGB(18, 4, 8),    Elevated = Color3.fromRGB(26, 8, 14) },
	Arctic      = { Accent = Color3.fromRGB(125, 211, 252), Surface = Color3.fromRGB(4, 10, 18),   Elevated = Color3.fromRGB(8, 16, 26) },
	Phantom     = { Accent = Color3.fromRGB(192, 132, 252), Surface = Color3.fromRGB(8, 4, 16),    Elevated = Color3.fromRGB(14, 8, 24) },
	Neon        = { Accent = Color3.fromRGB(255, 60, 200),  Surface = Color3.fromRGB(8, 2, 12),    Elevated = Color3.fromRGB(14, 4, 20) },
	Copper      = { Accent = Color3.fromRGB(200, 100, 40),  Surface = Color3.fromRGB(12, 6, 2),    Elevated = Color3.fromRGB(20, 10, 4) },
	Storm       = { Accent = Color3.fromRGB(100, 130, 200), Surface = Color3.fromRGB(6, 8, 16),    Elevated = Color3.fromRGB(10, 14, 24) },
}
local ThemeOrder = {
	"Midnight", "Gold", "Stealth", "Soft Pastel", "Red/Black",
	"Emerald", "Crimson", "Ocean", "Purple", "Sakura",
	"Cyber", "Obsidian", "Amber", "Lime", "Rose",
	"Arctic", "Phantom", "Neon", "Copper", "Storm",
}

-- Dark = Midnight surfaces | Light/White = white surfaces + accent
local ThemeMode = "Dark"
do
	local saved = _G.SosyThemeMode
	if saved == "Light" or saved == "White" then
		ThemeMode = "Light"
	elseif saved == "Dark" then
		ThemeMode = "Dark"
	else
		pcall(function()
			if isfile and isfile("SosyHUB/ThemeMode.txt") then
				local m = tostring(readfile("SosyHUB/ThemeMode.txt"))
				if m == "Light" or m == "White" then ThemeMode = "Light"
				elseif m == "Dark" then ThemeMode = "Dark" end
			end
		end)
	end
	_G.SosyThemeMode = ThemeMode
end
-- Force splash/login aesthetic to Stealth (no blue Midnight tint)
do
	local mp = ThemePresets.Stealth
	Theme.Accent = mp.Accent
	Theme.Surface = mp.Surface
	Theme.Elevated = mp.Elevated
	Theme.Header = mp.Elevated
	Theme.Hover = mp.Elevated:Lerp(mp.Accent, 0.12)
	Theme.Border = Color3.fromRGB(70, 70, 76)
	Theme.BorderStrong = Color3.fromRGB(100, 100, 108)
	Theme.Text = Color3.fromRGB(236, 240, 246)
	Theme.TextSecondary = Color3.fromRGB(168, 178, 194)
	Theme.TextMuted = Color3.fromRGB(110, 120, 138)
	Theme.ToggleOff = Color3.fromRGB(40, 46, 58)
	_G.SosyLastTheme = _G.SosyLastTheme or "Red/Black"
	pcall(function()
		if isfile and isfile("SosyHUB/LastTheme.txt") then
			_G.SosyLastTheme = tostring(readfile("SosyHUB/LastTheme.txt"))
		end
		if isfile and isfile("SosyHUB/LastDarkTheme.txt") then
			_G.SosyLastDarkTheme = tostring(readfile("SosyHUB/LastDarkTheme.txt"))
		end
	end)
	_G.SosyLastDarkTheme = _G.SosyLastDarkTheme or "Red/Black"
	_G.SosyLastLightTheme = nil -- light/dark is ThemeMode only; themes are accents
	-- migrate old light-only theme names → accent
	local migrate = {
		White = "Red/Black", ["White Mode"] = "Red/Black",
		["Gold on White"] = "Gold",
	}
	if migrate[_G.SosyLastTheme] then _G.SosyLastTheme = migrate[_G.SosyLastTheme] end
	if migrate[_G.SosyLastDarkTheme] then _G.SosyLastDarkTheme = migrate[_G.SosyLastDarkTheme] end
end

_G.SosyUILanguage = "en"

local function setThemeMode(mode)
	ThemeMode = (mode == "Light" or mode == "White") and "Light" or "Dark"
	_G.SosyThemeMode = ThemeMode
	pcall(function()
		if not isfolder("SosyHUB") then makefolder("SosyHUB") end
		writefile("SosyHUB/ThemeMode.txt", ThemeMode)
	end)
end

local function applyPreset(name)
	local p = ThemePresets[name]
	if not p then return end
	local accent = p.Accent
	Theme.Accent = accent
	if ThemeMode == "Light" then
		-- white / soft tint + accent (e.g. Violet â†’ white + purple)
		Theme.Surface = Color3.fromRGB(250, 251, 253)
		Theme.Elevated = Color3.fromRGB(255, 255, 255):Lerp(accent, 0.05)
		Theme.Header = Color3.fromRGB(245, 247, 251):Lerp(accent, 0.06)
		Theme.Hover = Theme.Elevated:Lerp(accent, 0.12)
		Theme.Border = Color3.fromRGB(220, 224, 234):Lerp(accent, 0.2)
		Theme.BorderStrong = Color3.fromRGB(198, 204, 220):Lerp(accent, 0.28)
		Theme.Text = Color3.fromRGB(22, 26, 34)
		Theme.TextSecondary = Color3.fromRGB(70, 78, 96)
		Theme.TextMuted = Color3.fromRGB(118, 126, 142)
		Theme.ToggleOff = Color3.fromRGB(228, 232, 240)
		Theme.Success = Color3.fromRGB(22, 163, 74)
		Theme.Error = Color3.fromRGB(220, 38, 38)
	else
		Theme.Surface = p.Surface
		Theme.Elevated = p.Elevated
		Theme.Header = p.Elevated
		Theme.Hover = p.Elevated:Lerp(accent, 0.12)
		Theme.Border = p.Surface:Lerp(accent, 0.25)
		Theme.BorderStrong = p.Surface:Lerp(accent, 0.4)
		Theme.Text = Color3.fromRGB(236, 240, 246)
		Theme.TextSecondary = Color3.fromRGB(168, 178, 194)
		Theme.TextMuted = Color3.fromRGB(110, 120, 138)
		Theme.ToggleOff = Color3.fromRGB(40, 46, 58)
		Theme.Success = Color3.fromRGB(74, 222, 128)
		Theme.Error = Color3.fromRGB(248, 113, 113)
	end
	_G.SosyLastTheme = name
	pcall(function()
		if not isfolder("SosyHUB") then makefolder("SosyHUB") end
		writefile("SosyHUB/LastTheme.txt", name)
	end)
end

-- Embedded feature/native modules from source 1.
local EMBED = {
    SosyBackend = [====[
--[[ SosyHUB Backend — Sedse-free feature state + natives + config + VPS API ]]
local HttpService = game:GetService("HttpService")
local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")

local Backend = {}
local State = {}
local Defaults = {}
local Api = {}

-- Backend is compiled as its own embedded chunk, so it cannot see NovaCore's
-- local ensure helper. Keep a local copy for every native state branch.
local function ensure(tblName, defaults)
	_G[tblName] = _G[tblName] or {}
	if type(defaults) == "table" then
		for k, v in pairs(defaults) do
			if _G[tblName][k] == nil then _G[tblName][k] = v end
		end
	end
	return _G[tblName]
end

local function cfg()
	_G.SosyConfig = _G.SosyConfig or {}
	local c = _G.SosyConfig
	c.PublicBase = tostring(c.PublicBase or _G.SosyPublicBase or "")
	c.AuthBase = tostring(c.AuthBase or _G.SosyDiscordAuthBase or c.PublicBase or "")
	c.Invite = tostring(c.Invite or _G.SosyDiscordInvite or "https://discord.gg/sosyhub")
	c.ScriptId = tostring(c.ScriptId or "sosyhub-bastion")
	return c
end

local function ensureFolder()
	pcall(function()
		if isfolder and not isfolder("SosyHUB") then makefolder("SosyHUB") end
		if isfolder and not isfolder("SosyHUB/configs") then makefolder("SosyHUB/configs") end
	end)
end

local function applyNative(name, value)
	local low = string.lower(tostring(name or ""))
	-- Blackflash / BFC
	if name == "Black Flash" or name == "Black Flash Chain" or name == "BFC" or name == "Enable Black Flash" then
		local v = value == true
		_G.NewBFCEnabled = v
		_G.BFC1Enabled = v
		_G.BlackFlashState = _G.BlackFlashState or {}
		_G.BlackFlashState.Enabled = v
		_G.BlackFlashChainEnabled = v
	elseif name == "BF Mobile Button" or name == "Show Mobile Dash Button" then
		_G.BFCMobileButtonEnabled = value == true
		if type(_G.SetBFCMobileButton) == "function" then
			pcall(_G.SetBFCMobileButton, value == true)
		end
	elseif name == "Detection Range" and type(value) == "number" then
		_G.BlackFlashState = _G.BlackFlashState or {}
		_G.BlackFlashState.DetectionRange = value
		_G.NewBFCRange = value
	elseif name == "BF Curve Strength" and type(value) == "number" then
		_G.BlackFlashState = _G.BlackFlashState or {}
		_G.BlackFlashState.CurveStrength = value
		_G.NewBFCCurveStrength = value
		_G.DashAssistArc = value
	elseif name == "Behind Radius (Distance)" and type(value) == "number" then
		_G.BlackFlashState = _G.BlackFlashState or {}
		_G.BlackFlashState.BehindRadius = value
		_G.NewBFCStopDistance = value
		_G.DashAssistBehind = value
	-- Dash Assist
	elseif name == "Enable Side Dash Assist" or name == "Side Dash Assist" then
		_G.DashAssistEnabled = value == true
		_G.OriginalSideDashEnabled = value == true
		_G.DashAssistState = _G.DashAssistState or {}
		_G.DashAssistState.Enabled = value == true
	elseif name == "Enable Mobile Button" then
		_G.DashAssistMobile = value == true
		_G.DashAssistState = _G.DashAssistState or {}
		_G.DashAssistState.MobileButtonEnabled = value == true
	elseif name == "Lock Camera On Enemy" then
		_G.DashAssistState = _G.DashAssistState or {}
		_G.DashAssistState.LockCamera = value == true
		_G.DashAssistState.CameraLock = value == true
	elseif name == "Dash Only If Facing Front" then
		_G.DashAssistState = _G.DashAssistState or {}
		_G.DashAssistState.FacingOnly = value == true
	elseif name == "Detection Distance" and type(value) == "number" then
		_G.DashAssistState = _G.DashAssistState or {}
		_G.DashAssistState.DetectionRange = value
		_G.DashAssistRange = value
	elseif name == "Behind Distance" and type(value) == "number" then
		_G.DashAssistState = _G.DashAssistState or {}
		_G.DashAssistState.BehindDistance = value
		_G.DashAssistBehind = value
	elseif name == "Flight Duration" or name == "Dash Duration (s)" then
		if type(value) == "number" then
			_G.DashAssistState = _G.DashAssistState or {}
			_G.DashAssistState.FlightDuration = value
			_G.DashAssistDuration = value
			_G.NewBFCDuration = value
		end
	elseif name == "Curve Strength" and type(value) == "number" then
		_G.DashAssistState = _G.DashAssistState or {}
		_G.DashAssistState.CurveStrength = value
		_G.DashAssistArc = value
	elseif name == "Arch Height" and type(value) == "number" then
		_G.DashAssistState = _G.DashAssistState or {}
		_G.DashAssistState.ArchHeight = value
	elseif name == "Lock Duration" and type(value) == "number" then
		_G.DashAssistState = _G.DashAssistState or {}
		_G.DashAssistState.LockDuration = value
	-- Auto Block
	elseif name == "Auto Block" then
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.Enabled = value == true
		_G.AutoBlockState.Trail = value == true
		_G.AutoBlockState.MoveAnim = value == true
		_G.AutoBlockState.Projectile = value == true
		if type(_G.SetAutoBlock) == "function" then pcall(_G.SetAutoBlock, value == true) end
	elseif low:find("trail block") then
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.Trail = value == true
		local st = _G.AutoBlockState
		st.Enabled = (st.Trail or st.MoveAnim or st.Projectile) == true
		if type(_G.SetAutoBlock) == "function" then pcall(_G.SetAutoBlock, st.Enabled) end
	elseif low:find("move anim block") then
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.MoveAnim = value == true
		local st = _G.AutoBlockState
		st.Enabled = (st.Trail or st.MoveAnim or st.Projectile) == true
		if type(_G.SetAutoBlock) == "function" then pcall(_G.SetAutoBlock, st.Enabled) end
	elseif low:find("projectile block") then
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.Projectile = value == true
		local st = _G.AutoBlockState
		st.Enabled = (st.Trail or st.MoveAnim or st.Projectile) == true
		if type(_G.SetAutoBlock) == "function" then pcall(_G.SetAutoBlock, st.Enabled) end
	elseif low:find("block while attacking") then
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.WhileAttacking = value == true
	-- Auto Block tuning sliders
	elseif name == "Trail Detection Distance" and type(value) == "number" then
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.TrailDistance = value
	elseif name == "Trail Ahead Angle" and type(value) == "number" then
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.TrailAngle = value
	elseif name == "Move Anim Detection Range" and type(value) == "number" then
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.MoveRange = value
	elseif name == "Move Facing Sensitivity" and type(value) == "number" then
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.MoveFacing = value
	elseif name == "Projectile Detection Range" and type(value) == "number" then
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.ProjectileRange = value
	elseif name == "Block Duration (s)" and type(value) == "number" then
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.BlockDuration = value
	-- Lock
	elseif name == "Enable Lock" then
		_G.LockState = _G.LockState or {}
		_G.LockState.Enabled = value == true
		if type(_G.SetLock) == "function" then pcall(_G.SetLock, value == true) end
	-- Character
	elseif name == "Anti-Stun / Knockback" then
		_G.AntiStun = value == true
		if type(_G.SetAntiStun) == "function" then pcall(_G.SetAntiStun, value == true) end
	elseif name == "Anti-Ragdoll" then
		_G.AntiRagdoll = value == true
		if type(_G.SetAntiRagdoll) == "function" then pcall(_G.SetAntiRagdoll, value == true) end
	elseif name == "Hitbox Expander" then
		_G.HitboxExpander = value == true
		_G.CharState = _G.CharState or {}
		_G.CharState.HitboxExpander = value == true
		if type(_G.SetHitboxExpander) == "function" then pcall(_G.SetHitboxExpander, value == true) end
	elseif name == "Hitbox Range" then
		_G.CharState = _G.CharState or {}
		_G.CharState.HitboxRangeOn = value == true
		if type(_G.SetHitboxRange) == "function" then pcall(_G.SetHitboxRange, value == true) end
	elseif name == "Hitbox Range Mult" and type(value) == "number" then
		_G.CharState = _G.CharState or {}
		_G.CharState.HitboxRange = value
	elseif name == "Hitbox Max Radius" and type(value) == "number" then
		_G.CharState = _G.CharState or {}
		_G.CharState.HitboxMaxRadius = value
	elseif name == "Hitbox Size" and type(value) == "number" then
		value = math.clamp(value, 1, 100)
		_G.HitboxSize = value
		_G.CharState = _G.CharState or {}
		_G.CharState.HitboxSize = value
	elseif name == "SosyHUB AC Bypass" then
		local on = value == true
		-- Primary: trigger through Sedse bridge → executes dump's "Start AC Bypass" handler
		local bridged = false
		pcall(function()
			if _G.SosyCore and type(_G.SosyCore.setSedse) == "function" then
				bridged = _G.SosyCore.setSedse("Start AC Bypass", on) == true
			end
		end)
		if not bridged then
			pcall(function()
				if type(_G.setSedse) == "function" then
					_G.setSedse("Start AC Bypass", on)
					bridged = true
				end
			end)
		end
		-- Fallback: built-in corner-teleport bypass
		if not bridged and type(_G.SosyHUBStartACBypass) == "function" then
			pcall(_G.SosyHUBStartACBypass, on)
		end
		pcall(function()
			if _G.SosyMacLibWindow then
				_G.SosyMacLibWindow:Notify({ Title = "SosyHUB AC Bypass", Description = on and "Starting bypass..." or "Stopped", Lifetime = 3 })
			end
		end)
	elseif name == "SosyHUB Rapid TP" then
		if type(_G.SosyHUBStartRapidTP) == "function" then
			pcall(_G.SosyHUBStartRapidTP, value == true)
		end
		pcall(function()
			if _G.SosyMacLibWindow then
				_G.SosyMacLibWindow:Notify({ Title = "SosyHUB Rapid TP", Description = value and "Rapid TP: ACTIVE" or "Rapid TP: stopped", Lifetime = 3 })
			end
		end)
	elseif name == "Location" then
		_G.TeleportState = _G.TeleportState or {}
		_G.TeleportState.Location = tostring(value)
	elseif name == "Avatar Username" then
		_G.AvatarSpoofUsername = tostring(value or "")
	elseif name == "Kills Spoof Username" then
		_G.KillsSpoofUsername = tostring(value or "")
		_G.KillsSpoofState = _G.KillsSpoofState or {}
		_G.KillsSpoofState.Username = tostring(value or "")
	elseif name == "Kills Spoof Amount" and type(value) == "number" then
		_G.KillsSpoofAmount = value
		_G.KillsSpoofState = _G.KillsSpoofState or {}
		_G.KillsSpoofState.Amount = value
		if _G.KillsSpoofState.Enabled and type(_G.ApplyKillsSpoof) == "function" then
			pcall(_G.ApplyKillsSpoof, value, true)
		end
	elseif name == "enable kills spoof" then
		_G.KillsSpoofState = _G.KillsSpoofState or {}
		_G.KillsSpoofState.Enabled = value == true
		_G.KillsSpoofState.Amount = tonumber(State["Kills Spoof Amount"]) or _G.KillsSpoofAmount or 0
		_G.KillsSpoofState.Username = tostring(State["Kills Spoof Username"] or _G.KillsSpoofUsername or "")
		if type(_G.ApplyKillsSpoof) == "function" then
			pcall(_G.ApplyKillsSpoof, _G.KillsSpoofState.Amount, value == true)
		end
	-- Misc
	elseif name == "Invisible" then
		_G.InvisibleEnabled = value == true
		_G.MiscState = _G.MiscState or {}
		_G.MiscState.IsInvisible = value == true
		if type(_G.SetInvisible) == "function" then pcall(_G.SetInvisible, value == true) end
	elseif name == "Infinite Dash" then
		_G.InfiniteDash = value == true
	elseif name == "Infinite Parkour" then
		-- SosyBackend is a separate embedded chunk; NovaCore's local ensure() is not
		-- visible here. Calling it used to throw before SetInfiniteParkour ran.
		_G.MiscState = _G.MiscState or {}
		_G.MiscState.InfiniteParkour = value == true
		if type(_G.SetInfiniteParkour) == "function" then
			pcall(_G.SetInfiniteParkour, value == true)
		end
	elseif name == "Noclip Domain" then
		_G.NoclipDomain = value == true
		if type(_G.SetNoclip) == "function" then pcall(_G.SetNoclip, value == true) end
	elseif name == "Flight" then
		_G.FlightEnabled = value == true
		if type(_G.SetFlight) == "function" then pcall(_G.SetFlight, value == true) end
	elseif name == "Base Speed" and type(value) == "number" then
		_G.FlightBaseSpeed = value
	elseif name == "Boost Speed (Shift)" and type(value) == "number" then
		_G.FlightBoostSpeed = value
	-- Block Break
	elseif name == "Enable Block Break" or name == "Block Break" then
		_G.BlockBreakEnabled = value == true
		_G.BlockBreakState = _G.BlockBreakState or {}
		_G.BlockBreakState.Enabled = value == true
		if type(_G.SetBlockBreak) == "function" then pcall(_G.SetBlockBreak, value == true) end
	-- Character specials
	elseif name == "Yuki Instant Charge" then
		_G.YukiInstantCharge = value == true
		_G.SpecialsState = _G.SpecialsState or {}
		_G.SpecialsState.YukiInstantCharge = value == true
	elseif name == "Ryu Silent Vent" then
		_G.RyuState = _G.RyuState or {}
		_G.RyuState.SilentVent = value == true
		-- Silent Vent is only an animation/SFX/recovery bypass. Keep the normal
		-- local Overheat/Buildup UI visible unless Hide Heat Bar is explicitly on.
		if value == true then
			_G.RyuState.LocalHeatBar = true
			_G.RyuState.HideHeatBar = false
			if type(hookRyuHeat) == "function" then pcall(hookRyuHeat) end
			if type(ensureRyuHeatLoop) == "function" then pcall(ensureRyuHeatLoop) end
		end
		if type(ensureRyuSilentVentLoop) == "function" then pcall(ensureRyuSilentVentLoop) end
	elseif name == "Auto QTE (Final Judgement)" then
		_G.AutoQTE = value == true
		_G.SpecialsState = _G.SpecialsState or {}
		_G.SpecialsState.HiguQTE = value == true
	elseif name == "Auto Ratio" then
		_G.AutoRatio = value == true
		if type(_G.SetAutoRatio) == "function" then pcall(_G.SetAutoRatio, value == true) end
	elseif name == "Auto Final Judgement" then
		_G.AutoFinalJudgement = value == true
		if type(_G.SetAutoFinalJudgement) == "function" then pcall(_G.SetAutoFinalJudgement, value == true) end
	elseif name == "Auto Skill Range" and type(value) == "number" then
		_G.CharState = _G.CharState or {}
		_G.CharState.AutoSkillRange = value
	elseif name == "No Dash Cooldowns" then
		_G.CharState = _G.CharState or {}
		_G.CharState.NoDashCD = value == true
		if type(_G.SetNoDashCD) == "function" then pcall(_G.SetNoDashCD, value == true) end
	elseif name == "Dash Reset Rate (ms)" and type(value) == "number" then
		_G.CharState = _G.CharState or {}
		_G.CharState.DashResetRate = value / 1000
	elseif name == "Auto Turn 180 on Swap" then
		_G.TodoState = _G.TodoState or {}
		_G.TodoState.AutoTurn = value == true
		if type(_G.SetAutoTurn) == "function" then pcall(_G.SetAutoTurn, value == true) end
	elseif name == "Auto Turn Hold (ms)" and type(value) == "number" then
		_G.TodoState = _G.TodoState or {}
		_G.TodoState.AutoTurnHold = value / 1000
	elseif name == "0.2 Domain Farm" then
		_G.SpecialsState = _G.SpecialsState or {}
		_G.SpecialsState.Domain02Farm = value == true
		if type(_G.SetDomain02Farm) == "function" then pcall(_G.SetDomain02Farm, value == true) end
	elseif name == "Auto Buy Soda When Low" or name == "Auto Soda On Low Health" then
		_G.MiscState = _G.MiscState or {}
		_G.MiscState.AutoBuySoda = value == true
		if type(_G.SetAutoBuySoda) == "function" then pcall(_G.SetAutoBuySoda, value == true) end
	elseif name == "Soda Buy HP %" and type(value) == "number" then
		_G.MiscState = _G.MiscState or {}
		_G.MiscState.SodaBuyHpPct = value
	elseif name == "Auto Perfect Clap" then
		_G.TodoState = _G.TodoState or {}
		_G.TodoState.AutoPerfectClap = value == true
		if type(_G.SetAutoPerfectClap) == "function" then pcall(_G.SetAutoPerfectClap, value == true) end
	elseif name == "Perfect Clap Look (ms)" and type(value) == "number" then
		_G.TodoState = _G.TodoState or {}
		_G.TodoState.PerfectClapLook = value / 1000
	elseif name == "Perfect Clap Delay (ms)" and type(value) == "number" then
		_G.TodoState = _G.TodoState or {}
		_G.TodoState.PerfectClapDelay = value / 1000
	elseif name == "Fast Swap (Predict)" then
		_G.TodoState = _G.TodoState or {}
		_G.TodoState.FastSwap = value == true
		if type(_G.SetTodoFastSwap) == "function" then pcall(_G.SetTodoFastSwap, value == true) end
	elseif name == "Fast Swap Offset" and type(value) == "number" then
		_G.TodoState = _G.TodoState or {}
		_G.TodoState.FastSwapOffset = value
	elseif name == "Fast Clap Anim (visual)" then
		_G.TodoState = _G.TodoState or {}
		_G.TodoState.FastSwapAnim = value == true
		if type(_G.SetTodoFastSwapAnim) == "function" then pcall(_G.SetTodoFastSwapAnim, value == true) end
	elseif name == "Clap Anim Speed" and type(value) == "number" then
		_G.TodoState = _G.TodoState or {}
		_G.TodoState.FastSwapAnimSpeed = value
	elseif name == "Inf Ult" then
		_G.MiscState = _G.MiscState or {}
		_G.MiscState.InfUlt = value == true
		if type(_G.SetInfUlt) == "function" then pcall(_G.SetInfUlt, value == true) end
	elseif name == "Inf Ult TP Duration (s)" and type(value) == "number" then
		_G.MiscState = _G.MiscState or {}
		_G.MiscState.InfUltBurst = value
	elseif name == "Inf Ult Lead (ms)" or name == "Inf Ult TP Range" then
		-- retired: Inf Ult now bursts after the ult ends instead of leading into it.
		-- Swallowed so an older saved config does not fall through to the else branch.
	elseif name == "Higuruma Vote Viewer" then
		_G.MiscState = _G.MiscState or {}
		_G.MiscState.HiromiVotes = value == true
		if type(_G.SetHiromiVotes) == "function" then pcall(_G.SetHiromiVotes, value == true) end
	elseif name == "Anti Domain" then
		_G.MiscState = _G.MiscState or {}
		_G.MiscState.AntiDomain = value == true
		if type(_G.SetAntiDomain) == "function" then pcall(_G.SetAntiDomain, value == true) end
	elseif name == "Instant Blackhole" then
		_G.SpecialsState = _G.SpecialsState or {}
		_G.SpecialsState.InstantBlackhole = value == true
		if type(_G.SetInstantBlackhole) == "function" then pcall(_G.SetInstantBlackhole, value == true) end
	elseif name == "Cooldown Viewer" then
		_G.MiscState = _G.MiscState or {}
		_G.MiscState.CooldownViewer = value == true
		if type(_G.SetCooldownViewer) == "function" then pcall(_G.SetCooldownViewer, value == true) end
	elseif name == "ESP" or name == "Character ESP" then
		_G.CharacterESP = value == true
		if type(_G.SetESP) == "function" then pcall(_G.SetESP, value == true) end
	elseif name == "Player Name ESP" then
		_G.PlayerNameESP = value == true
		if type(_G.SetNameESP) == "function" then pcall(_G.SetNameESP, value == true) end
	elseif name == "Piano Enabled" then
		_G.PianoState = _G.PianoState or {}
		_G.PianoState.Enabled = value == true
	elseif name == "Piano Speed" and type(value) == "number" then
		_G.PianoState = _G.PianoState or {}
		_G.PianoState.Speed = value
	elseif name == "Piano Transpose" and type(value) == "number" then
		_G.PianoState = _G.PianoState or {}
		_G.PianoState.Transpose = value
	elseif name == "Piano Loop" then
		_G.PianoState = _G.PianoState or {}
		_G.PianoState.Looping = value == true
	elseif name == "Song" then
		_G.PianoState = _G.PianoState or {}
		local songName = tostring(value)
		_G.PianoState.FilePath = songName
		if type(_G.pianoLoadSong) == "function" and value and songName ~= "None" and not _G.PianoState.Loading then
			-- Parsing yields and can take a while on large files, so it must not run
			-- inline on the dropdown callback — that is what locked up the game.
			_G.PianoState.Loading = true
			task.spawn(function()
				if type(_G.pianoStopPlayback) == "function" then pcall(_G.pianoStopPlayback) end
				local called, ok, info = pcall(_G.pianoLoadSong, songName)
				_G.PianoState.Loading = false
				pcall(function()
					if not (_G.SosyMacLibWindow and _G.SosyMacLibWindow.Notify) then return end
					local good = called and ok
					_G.SosyMacLibWindow:Notify({
						Title = "Piano",
						Description = good
							and (songName .. " loaded (" .. tostring(info) .. " notes)")
							or ("Could not load " .. songName .. " — " .. tostring(called and info or "parse error")),
						Lifetime = 4,
					})
				end)
			end)
		end
	elseif name == "AI Assistant" then
		_G.SosyAIEnabled = value == true
	elseif name == "AI Rebrand TBO Text" then
		_G.SosyAIRebrand = value == true
	elseif name == "Ryu Hide Heat Bar" then
		_G.RyuState = _G.RyuState or {}
		_G.RyuState.HideHeatBar = value == true
		if type(ensureRyuHeatLoop) == "function" then pcall(ensureRyuHeatLoop) end
	elseif name == "Ryu Local Heat Bar" then
		_G.RyuState = _G.RyuState or {}
		_G.RyuState.LocalHeatBar = value == true
		if type(ensureRyuHeatLoop) == "function" then pcall(ensureRyuHeatLoop) end
	elseif name == "Todo Bring Range" and type(value) == "number" then
		_G.SpecialsState = _G.SpecialsState or {}
		_G.SpecialsState.TodoBringRange = value
	elseif _G.SosyShaders and type(_G.SosyShaders.isShaderControl) == "function" and _G.SosyShaders.isShaderControl(name) then
		pcall(function() _G.SosyShaders.applyControl(name, value) end)
	elseif name == "Camera FOV" and type(value) == "number" then
		if _G.SosyShaders then pcall(function() _G.SosyShaders.setCameraFov(value) end) end
	elseif name == "Select Theme" then
		local v = tostring(value)
		pcall(applyPreset, v)
		if type(_G.SosyApplyTheme) == "function" then pcall(_G.SosyApplyTheme, v) end
	elseif name == "UI Font" then
		local fontMap = {
			["GothamBold"]    = Enum.Font.GothamBold,
			["Gotham"]        = Enum.Font.Gotham,
			["SourceSansPro"] = Enum.Font.SourceSansPro,
			["RobotoMono"]    = Enum.Font.RobotoMono,
			["Oswald"]        = Enum.Font.Oswald,
			["SciFi"]         = Enum.Font.SciFi,
			["Fantasy"]       = Enum.Font.Fantasy,
			["Cartoon"]       = Enum.Font.Cartoon,
			["Code"]          = Enum.Font.Code,
			["Bodoni"]        = Enum.Font.Bodoni,
		}
		local fnt = fontMap[tostring(value)]
		if fnt then
			_G.SosyUIFont = fnt
			pcall(function()
				local guiRoot
				pcall(function() if type(gethui) == "function" then guiRoot = gethui() end end)
				guiRoot = guiRoot or game:GetService("CoreGui")
				local function applyFont(root)
					for _, d in ipairs(root:GetDescendants()) do
						if (d:IsA("TextLabel") or d:IsA("TextButton") or d:IsA("TextBox")) and not d:GetAttribute("SosyNoFont") then
							pcall(function() d.Font = fnt end)
						end
					end
				end
				local sosyGui = guiRoot:FindFirstChild("SosyMacLib")
				if sosyGui then applyFont(sosyGui) end
				local softUI = guiRoot:FindFirstChild("SosyHUBSoftUI")
				if softUI then applyFont(softUI) end
			end)
			api.notify("Font: " .. tostring(value))
		end
	else
		_G.SosyFeatureState = _G.SosyFeatureState or {}
		_G.SosyFeatureState[name] = value
	end
end

local function runButton(name)
	local lp = Players.LocalPlayer
	if name == "Force Reset" then
		pcall(function()
			local char = lp.Character
			if char then
				local hum = char:FindFirstChildOfClass("Humanoid")
				if hum then hum.Health = 0 end
			end
		end)
	elseif name == "Teleport" then
		local loc = State["Location"] or "Bowling Alley"
		local ok, res = pcall(function()
			if type(_G.SosyTeleportTo) == "function" then
				return _G.SosyTeleportTo(loc)
			end
			-- fallback: direct CFrame teleport via workspace scan
			local lp = game:GetService("Players").LocalPlayer
			local char = lp.Character or lp.CharacterAdded:Wait(3)
			local hrp = char and char:FindFirstChild("HumanoidRootPart")
			if not hrp then return false end
			local function scanForPart(root)
				for _, d in ipairs(root:GetDescendants()) do
					if (d:IsA("BasePart") or d:IsA("SpawnLocation")) and
						string.lower(d.Name):find(string.lower(loc), 1, true) then
						return d
					end
				end
			end
			local target = scanForPart(workspace)
			if target then
				hrp.CFrame = target.CFrame + Vector3.new(0, 4, 0)
				return true
			end
			return false
		end)
		pcall(function()
			if _G.SosyMacLibWindow then
				if ok and res then
					_G.SosyMacLibWindow:Notify({ Title = "Teleport", Description = "Teleported to " .. loc, Lifetime = 3 })
				else
					_G.SosyMacLibWindow:Notify({ Title = "Teleport Failed", Description = "Could not find: " .. loc, Lifetime = 4 })
				end
			end
		end)
	elseif name == "Gojo 0.2 Ultimate" then
		_G.OPKillsState = _G.OPKillsState or {}
		_G.OPKillsState.IsGojoUltimate = true
		if type(_G.StartGojoUltimate) == "function" then pcall(_G.StartGojoUltimate) end
		pcall(function()
			_G.SosyStartRapidKillAll(true)
		end)
	elseif name == "apply avatar" then
		local user = tostring(State["Avatar Username"] or _G.AvatarSpoofUsername or "")
		if user ~= "" and type(_G.ApplyAvatarSpoof) == "function" then
			pcall(_G.ApplyAvatarSpoof, user)
		elseif user ~= "" then
			_G.AvatarSpoofUsername = user
			_G.AvatarSpoofPending = user
		end
	elseif name == "Refresh Items" then
		local opts = { "None" }
		local seen = {}
		pcall(function()
			local folder = workspace:FindFirstChild("Items")
			if folder then
				for _, obj in ipairs(folder:GetChildren()) do
					if not seen[obj.Name] then
						seen[obj.Name] = true
						table.insert(opts, obj.Name)
					end
				end
			end
		end)
		Backend.setOptions("Select Item", opts)
		api.notify("Items: " .. (#opts - 1) .. " found")
	elseif name == "Grab Item" then
		local item = tostring(State["Select Item"] or "None")
		if item ~= "None" and item ~= "" then
			pcall(function()
				local char = lp.Character
				local hrp  = char and char:FindFirstChild("HumanoidRootPart")
				if not hrp then return end
				local folder = workspace:FindFirstChild("Items")
				if not folder then return end
				for _, obj in ipairs(folder:GetChildren()) do
					if obj.Name == item then
						local part = obj:IsA("BasePart") and obj or obj:FindFirstChildOfClass("BasePart")
						local prompt = obj:FindFirstChildWhichIsA("ProximityPrompt", true)
						if not part then
							api.notify("Item has no part: " .. item)
							return
						end
						-- Blink to the item, fire its real pickup prompt (so it goes
						-- through the game's own inventory flow instead of a fake local
						-- grab), then blink straight back. Not a lingering teleport --
						-- the trip there and back happens inside ~0.1s.
						local originalCFrame = hrp.CFrame
						hrp.CFrame = CFrame.new(part.Position + Vector3.new(0, 3, 0))
						if prompt then
							if type(fireproximityprompt) == "function" then
								fireproximityprompt(prompt)
							else
								prompt:InputHoldBegin()
								prompt:InputHoldEnd()
							end
						end
						task.wait(0.1)
						if hrp and hrp.Parent then hrp.CFrame = originalCFrame end
						return
					end
				end
				api.notify("Item not found: " .. item)
			end)
		end
	elseif name == "Refresh Player List" or name == "Update Player List" or name == "Refresh Targets" or name == "Refresh Utility Targets" then
		local names = {"None", "Dummy"}
		for _, pl in ipairs(Players:GetPlayers()) do
			if pl ~= lp then table.insert(names, pl.Name) end
		end
		Backend.setOptions("Exclude Players List", names)
		Backend.setOptions("Select Target", names)
		Backend.setOptions("Select Utility Target", names)
		Backend.setOptions("Choose Attacker", names)
		Backend.setOptions("Todo Bring Target", names)
	elseif name == "Save Config" then
		local nm = tostring(State["Config Name"] or ""):gsub("^%s+", ""):gsub("%s+$", "")
		if nm == "" then nm = tostring(State["Saved Configs"] or "") end
		if nm == "" or nm == "None" then nm = "default" end
		-- sanitize: keep filesystem-safe characters only
		nm = nm:gsub("[^%w%-_ ]", ""):gsub("%s+", " "):gsub("^%s+", ""):gsub("%s+$", "")
		if nm == "" then nm = "default" end
		Backend.saveConfig(nm)
		pcall(function()
			if _G.SosyMacLibWindow then
				_G.SosyMacLibWindow:Notify({ Title = "Config", Description = "Saved: " .. nm, Lifetime = 3 })
			end
		end)
	elseif name == "Load Config" then
		Backend.loadConfig(tostring(State["Saved Configs"] or "default"))
	elseif name == "Delete Config" then
		pcall(function()
			local n = tostring(State["Saved Configs"] or "")
			if n ~= "" and n ~= "None" and delfile and isfile("SosyHUB/configs/" .. n .. ".json") then
				delfile("SosyHUB/configs/" .. n .. ".json")
			end
		end)
		Backend.refreshConfigList()
	elseif name == "Refresh Config List" then
		Backend.refreshConfigList()
	elseif name == "Refresh Songs" or name == "Scan Songs" then
		if type(_G.pianoScanSongs) == "function" then
			task.spawn(function()
				local list = _G.pianoScanSongs() or {}
				local opts = { "None" }
				for _, s in ipairs(list) do
					table.insert(opts, tostring(s))
				end
				Backend.setOptions("Song", opts)
				local n = math.max(0, #opts - 1)
				if n == 0 and type(_G.pianoScanReport) == "function" then
					-- Nothing found: dump what the scanner actually saw so the cause is
					-- visible instead of guessed.
					pcall(_G.pianoScanReport)
				end
				pcall(function()
					if _G.SosyMacLibWindow and _G.SosyMacLibWindow.Notify then
						local desc
						if n > 0 then
							desc = tostring(n) .. " MIDI found — open Song dropdown"
						else
							local d = (_G.PianoState and _G.PianoState.ScanDebug) or {}
							desc = "0 MIDI — saw " .. tostring(d.entries or 0) .. " files in "
								.. tostring(#(d.listed or {})) .. " folders. See console (F9) for the scan report."
						end
						_G.SosyMacLibWindow:Notify({
							Title = "Piano",
							Description = desc,
							Lifetime = 6,
						})
					end
				end)
			end)
		end
	elseif name == "Play Song" then
		if type(_G.pianoStartPlayback) == "function" then
			-- Play used to fail silently (no song selected, load error, still parsing).
			-- Surface the reason instead of leaving a dead button.
			task.spawn(function()
				local called, started = pcall(_G.pianoStartPlayback)
				if called and started then return end
				local why = (_G.PianoState and _G.PianoState.LastError) or "unknown error"
				pcall(function()
					if _G.SosyMacLibWindow and _G.SosyMacLibWindow.Notify then
						_G.SosyMacLibWindow:Notify({
							Title = "Piano",
							Description = "Could not play — " .. tostring(why),
							Lifetime = 5,
						})
					end
				end)
			end)
		end
	elseif name == "Pause Song" then
		if type(_G.pianoPausePlayback) == "function" then
			pcall(_G.pianoPausePlayback)
		end
	elseif name == "Stop Song" then
		if type(_G.pianoStopPlayback) == "function" then
			pcall(_G.pianoStopPlayback)
		end
	elseif name == "Refresh AI Chrome" then
		pcall(function()
			if type(_G._SosyTboScanCleanup) == "function" then
				-- trigger rebrand by notifying
			end
		end)
	elseif _G.SosyShaders and type(_G.SosyShaders.isShaderControl) == "function" and _G.SosyShaders.isShaderControl(name) then
		pcall(function() _G.SosyShaders.click(name) end)
	end
end

function Backend.initDefaults(catalog)
	for _, sec in ipairs(catalog or {}) do
		for _, it in ipairs(sec.items or {}) do
			local n = it.n
			if it.t == "toggle" then
				Defaults[n] = false
				State[n] = false
			elseif it.t == "slider" then
				Defaults[n] = tonumber(it.cur) or tonumber(it.min) or 0
				State[n] = Defaults[n]
			elseif it.t == "dropdown" or it.t == "list" then
				Defaults[n] = it.cur or ((it.opts and it.opts[1]) or "None")
				State[n] = Defaults[n]
				Api[n] = Api[n] or {}
				Api[n]._opts = it.opts or { "None" }
			elseif it.t == "status" then
				State[n] = n
			elseif it.t == "textbox" then
				Defaults[n] = it.cur or ""
				State[n] = Defaults[n]
			end
		end
	end
end

function Backend.get(name)
	return State[name]
end

function Backend.set(name, value)
	State[name] = value
	applyNative(name, value)
	local a = Api[name]
	if a and type(a._uiSet) == "function" then
		pcall(a._uiSet, value)
	end
	return true
end

function Backend.click(name)
	runButton(name)
	return true
end

function Backend.setOptions(name, opts)
	Api[name] = Api[name] or {}
	Api[name]._opts = opts or {}
	if type(Api[name]._uiSetOptions) == "function" then
		pcall(Api[name]._uiSetOptions, opts)
	end
end

function Backend.bindUi(name, handlers)
	Api[name] = Api[name] or {}
	for k, v in pairs(handlers or {}) do
		Api[name][k] = v
	end
end

function Backend.findApi(name)
	Api[name] = Api[name] or {}
	local a = Api[name]
	a.enabled = State[name] == true
	a.set = function(_, v)
		Backend.set(name, v)
	end
	a.set_value = function(_, v)
		Backend.set(name, v)
	end
	a.get_value = function()
		return State[name]
	end
	a.set_items = function(_, opts)
		Backend.setOptions(name, opts)
	end
	return a
end

function Backend.saveConfig(name)
	ensureFolder()
	name = tostring(name or "default")
	pcall(function()
		writefile("SosyHUB/configs/" .. name .. ".json", HttpService:JSONEncode(State))
	end)
	Backend.refreshConfigList()
end

function Backend.loadConfig(name)
	name = tostring(name or "default")
	pcall(function()
		if isfile and isfile("SosyHUB/configs/" .. name .. ".json") then
			local data = HttpService:JSONDecode(readfile("SosyHUB/configs/" .. name .. ".json"))
			if type(data) == "table" then
				for k, v in pairs(data) do
					Backend.set(k, v)
				end
			end
		end
	end)
end

function Backend.refreshConfigList()
	-- Start from the script-embedded list. Local files are never required.
	local opts = {}
	for _, name in ipairs(CONFIG_LIST or {}) do
		table.insert(opts, tostring(name))
	end

	-- If a real executor filesystem is available, append its configs too.
	-- The single-file build itself never depends on them.
	pcall(function()
		if listfiles and isfolder and isfolder("SosyHUB/configs") then
			for _, f in ipairs(listfiles("SosyHUB/configs")) do
				local base = tostring(f):match("([^/\\]+)%.json$")
				if base and base ~= "default" then
					local exists = false
					for _, v in ipairs(opts) do if v == base then exists = true break end end
					if not exists then table.insert(opts, base) end
				end
			end
		end
	end)

	if #opts == 0 then opts = { "default" } end
	Backend.setOptions("Saved Configs", opts)
	Backend.setOptions("Auto Load Config", opts)
end

function Backend.httpGet(path)
	local c = cfg()
	local base = c.PublicBase
	if base == "" then return nil, "no PublicBase" end
	local url = base:gsub("/$", "") .. "/" .. tostring(path or ""):gsub("^/", "")
	local ok, body = pcall(function()
		return game:HttpGet(url)
	end)
	if not ok then return nil, body end
	return body
end

function Backend.bootstrapFromVps()
	local c = cfg()
	if c.PublicBase == "" then return false end
	local body = Backend.httpGet("public/sosy-config.json")
	if type(body) == "string" and #body > 2 then
		pcall(function()
			local data = HttpService:JSONDecode(body)
			if type(data) == "table" then
				for k, v in pairs(data) do
					c[k] = v
				end
			end
		end)
		return true
	end
	return false
end

-- Dump loader: DISABLED.
-- The Potassium source dump (SOURCE/004_src.lua, 006_src.lua) contains encrypted
-- Luarmor V4 loaders. Executing them throws the Luarmor "loader code is outdated"
-- Auth Error and crashes the game. This build is self-contained — NovaCore already
-- provides every feature engine — so loading the dump is unnecessary and harmful.
-- Kept as a no-op so any caller stays safe.
_G.SosyLoadDump = function() end

-- Natives (Sedse-compat flags + lightweight loops)
_G.SetACBypass = function(on)
	on = on == true
	_G.ACBypassState = _G.ACBypassState or {}
	_G.ACBypassState.Enabled = on
	_G.ACBypassState.Status = on and "Status: anti cheat bypassed" or "Status: anti cheat not bypassed"
	_G.AutoKillState = _G.AutoKillState or {}
	_G.AutoKillState.ACBypass = on
	_G.KillAllState = _G.KillAllState or {}
	_G.KillAllState.ACBypass = on
	pcall(function()
		if typeof(getgenv) == "function" then
			local g = getgenv()
			g.ACBypassState = _G.ACBypassState
			g.AutoKillState = _G.AutoKillState
			g.KillAllState = _G.KillAllState
		end
	end)
	-- Namecall shield — try hookmetamethod first, fall back to getrawmetatable
	if on and not _G._SosyACHooked then
		pcall(function()
			local function makeHandler(callOld)
				local gmn = rawget(_G, "getnamecallmethod")
				local h = function(self, ...)
					if _G.ACBypassState and _G.ACBypassState.Enabled then
						local m = ""
						if typeof(gmn) == "function" then pcall(function() m = string.lower(tostring(gmn())) end) end
						if m == "kick" then return end
						if m == "fireserver" or m == "invokeserver" then
							local n = string.lower(tostring(typeof(self) == "Instance" and self.Name or ""))
							if n:find("kick") or n:find("ban") or n:find("anticheat") or n:find("exploit") or n:find("detect") then return end
						end
					end
					return callOld(self, ...)
				end
				if typeof(newcclosure) == "function" then h = newcclosure(h) end
				return h
			end
			-- Method 1: hookmetamethod (Synapse / Krnl style)
			if typeof(hookmetamethod) == "function" and typeof(getnamecallmethod) == "function" then
				_G._SosyACHooked = true
				local old; old = hookmetamethod(game, "__namecall", makeHandler(function(s,...) return old(s,...) end))
				return
			end
			-- Method 2: getrawmetatable (Potassium / Fluxus / Delta style)
			if typeof(getrawmetatable) == "function" then
				local mt = getrawmetatable(game)
				if type(mt) ~= "table" then return end
				local oldNC = rawget(mt, "__namecall")
				if typeof(oldNC) ~= "function" then return end
				local sro = rawget(_G,"setreadonly") or rawget(_G,"make_writeable")
				if sro then pcall(sro, mt, false) end
				_G._SosyACHooked = true
				rawset(mt, "__namecall", makeHandler(function(s,...) return oldNC(s,...) end))
			end
		end)
	end
end

_G.SosyStartRapidKillAll = function(on)
	_G.KillAllState = _G.KillAllState or {}
	_G.KillAllState.Gojo02 = on == true
	if _G._SosyRapidKillConn then
		pcall(function()
			_G._SosyRapidKillConn:Disconnect()
		end)
		_G._SosyRapidKillConn = nil
	end
	if on ~= true then return end
	local idx = 1
	local acc = 0
	_G._SosyRapidKillConn = game:GetService("RunService").Heartbeat:Connect(function(dt)
		if not (_G.KillAllState and _G.KillAllState.Gojo02) then return end
		acc += dt
		if acc < 0.1 then return end
		acc = 0
		local me = Players.LocalPlayer
		local myChar = me.Character
		local myHrp = myChar and myChar:FindFirstChild("HumanoidRootPart")
		if not myHrp then return end
		local list = {}
		for _, pl in ipairs(Players:GetPlayers()) do
			if pl ~= me and pl.Character and pl.Character:FindFirstChild("HumanoidRootPart") then
				local hum = pl.Character:FindFirstChildOfClass("Humanoid")
				if hum and hum.Health > 0 then
					table.insert(list, pl.Character.HumanoidRootPart)
				end
			end
		end
		if #list == 0 then return end
		idx = ((idx - 1) % #list) + 1
		local t = list[idx]
		idx += 1
		pcall(function()
			myHrp.CFrame = t.CFrame * CFrame.new(0, 0, 2.2)
		end)
	end)
end

-- SosyHUB AC Bypass (logic from xrest.lua, GUI stripped)
_G.SosyHUBStartACBypass = function(on)
	_G.SosyHUBACBypassState = _G.SosyHUBACBypassState or {}
	if not on then
		_G.SosyHUBACBypassState.Running = false
		pcall(function()
			if _G.SosyBackend then _G.SosyBackend.set("Status: SosyHUB AC Idle", "Idle") end
		end)
		return
	end
	if _G.SosyHUBACBypassState.Running then return end
	_G.SosyHUBACBypassState.Running = true

	task.spawn(function()
		local Players = game:GetService("Players")
		local RS = game:GetService("ReplicatedStorage")
		local me = Players.LocalPlayer

		local ResetRemote
		pcall(function()
			ResetRemote = RS.Knit.Knit.Services.JoinService.RE.Reset
		end)

		local function setStatus(txt)
			pcall(function()
				if _G.SosyBackend then _G.SosyBackend.set("Status: SosyHUB AC Idle", txt) end
			end)
		end

		local function getCF()
			local c = me.Character
			return c and c:FindFirstChild("HumanoidRootPart") and c.HumanoidRootPart.CFrame
		end

		local function isRubberBanded(origCF)
			local cur = getCF()
			if not cur then return true end
			return (cur.Position - origCF.Position).Magnitude < 8
		end

		local function knitReset()
			if ResetRemote then pcall(function() ResetRemote:FireServer() end) end
		end

		local function waitRespawn()
			local old = me.Character
			local t = 0
			while me.Character == old and t < 10 do task.wait(0.1); t += 0.1 end
			local new = me.Character
			if not new or not new.Parent or new == old then new = me.CharacterAdded:Wait() end
			new:WaitForChild("HumanoidRootPart", 10)
			task.wait(0.3)
		end

		local corners = {
			CFrame.new(349.089, 61.672, -344.117, -0.987, -0.000, 0.162, -0.000, 1.000, -0.000, -0.162, -0.000, -0.987),
			CFrame.new(-349.089, 61.672, -344.117),
			CFrame.new(349.089, 61.672, 344.117),
			CFrame.new(-349.089, 61.672, 344.117),
		}

		local safePart = Instance.new("Part")
		safePart.Size = Vector3.new(30, 2, 30)
		safePart.Anchored = true
		safePart.Transparency = 1
		safePart.CanCollide = true
		safePart.Parent = workspace

		local cornerIdx, attempts = 1, 0
		while _G.SosyHUBACBypassState and _G.SosyHUBACBypassState.Running do
			local origCF = getCF()
			if not origCF then task.wait(1); continue end

			attempts += 1
			local target = corners[cornerIdx]
			safePart.CFrame = target * CFrame.new(0, -4, 0)
			setStatus("Attempt " .. attempts .. " → Corner " .. cornerIdx)

			local char = me.Character
			if char and char:FindFirstChild("HumanoidRootPart") then
				char.HumanoidRootPart.CFrame = target
			end
			task.wait(0.35)

			if isRubberBanded(origCF) then
				setStatus("Rubber-banded, resetting...")
				knitReset()
				waitRespawn()
				task.wait(0.3)
				cornerIdx = (cornerIdx % #corners) + 1
			else
				setStatus("AC Bypassed!")
				pcall(function() safePart:Destroy() end)
				task.wait(0.5)
				local char2 = me.Character
				if char2 and char2:FindFirstChild("HumanoidRootPart") then
					char2.HumanoidRootPart.CFrame = origCF
				end
				_G.SosyHUBACBypassState.Running = false
				-- auto-flip toggle off in UI
				pcall(function()
					if _G.SosyBackend then _G.SosyBackend.set("SosyHUB AC Bypass", false) end
				end)
				return
			end
		end
		pcall(function() safePart:Destroy() end)
		setStatus("Idle")
	end)
end

-- SosyHUB Rapid TP (players + NPCs)
_G.SosyHUBStartRapidTP = function(on)
	_G.SosyHUBRapidTPState = _G.SosyHUBRapidTPState or {}
	_G.SosyHUBRapidTPState.Running = on == true
	if not on then return end
	task.spawn(function()
		local Players = game:GetService("Players")
		local me = Players.LocalPlayer
		while _G.SosyHUBRapidTPState and _G.SosyHUBRapidTPState.Running do
			local myChar = me.Character
			local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
			if myHRP then
				-- 1) Players
				for _, p in ipairs(Players:GetPlayers()) do
					if not (_G.SosyHUBRapidTPState and _G.SosyHUBRapidTPState.Running) then break end
					if p ~= me and p.Character then
						local thrp = p.Character:FindFirstChild("HumanoidRootPart")
						if thrp then
							myHRP.CFrame = thrp.CFrame * CFrame.new(0, 0, 3)
							task.wait(0.05)
						end
					end
				end
				-- 2) NPCs (workspace descendants with Humanoid alive, not a player character)
				if _G.SosyHUBRapidTPState and _G.SosyHUBRapidTPState.Running then
					local playerChars = {}
					for _, p in ipairs(Players:GetPlayers()) do
						if p.Character then playerChars[p.Character] = true end
					end
					for _, hum in ipairs(workspace:GetDescendants()) do
						if not (_G.SosyHUBRapidTPState and _G.SosyHUBRapidTPState.Running) then break end
						if hum:IsA("Humanoid") and hum.Health > 0 then
							local npcChar = hum.Parent
							if npcChar and not playerChars[npcChar] then
								local thrp = npcChar:FindFirstChild("HumanoidRootPart")
								if thrp then
									myHRP.CFrame = thrp.CFrame * CFrame.new(0, 0, 3)
									task.wait(0.05)
								end
							end
						end
					end
				end
			end
			task.wait(0.1)
		end
	end)
end

-- ─────────────────────────────────────────────────────────────────────────────
-- SosyHUB Feature Engine — implements all _G.Set* functions
-- ─────────────────────────────────────────────────────────────────────────────
do
	local RS  = game:GetService("RunService")
	local Plrs = game:GetService("Players")
	local me   = Plrs.LocalPlayer

	local function getChar()  return me.Character end
	local function getHRP()
		local c = getChar(); return c and c:FindFirstChild("HumanoidRootPart")
	end

	-- ── LOCK ──────────────────────────────────────────────────────────────────
	local _lockConn
	_G.SetLock = function(on)
		if _lockConn then _lockConn:Disconnect(); _lockConn = nil end
		if not on then return end
		_lockConn = RS.RenderStepped:Connect(function()
			if not (_G.LockState and _G.LockState.Enabled) then return end
			local hrp = getHRP(); if not hrp then return end
			local best, bd = nil, math.huge
			for _, pl in ipairs(Plrs:GetPlayers()) do
				if pl ~= me and pl.Character then
					local h = pl.Character:FindFirstChild("HumanoidRootPart")
					local hum = pl.Character:FindFirstChildOfClass("Humanoid")
					if h and hum and hum.Health > 0 then
						local d = (hrp.Position - h.Position).Magnitude
						if d < bd then bd = d; best = h end
					end
				end
			end
			if best then
				local cam = workspace.CurrentCamera
				cam.CFrame = CFrame.lookAt(cam.CFrame.Position, best.Position)
			end
		end)
	end

	-- ── FLIGHT ────────────────────────────────────────────────────────────────
	local _flightConn, _flightBV
	_G.SetFlight = function(on)
		if _flightConn then _flightConn:Disconnect(); _flightConn = nil end
		if _flightBV then pcall(function() _flightBV:Destroy() end); _flightBV = nil end
		if not on then return end
		_flightConn = RS.Heartbeat:Connect(function()
			if not _G.FlightEnabled then return end
			local char = getChar(); if not char then return end
			local hrp = char:FindFirstChild("HumanoidRootPart"); if not hrp then return end
			local hum = char:FindFirstChildOfClass("Humanoid")
			if hum then hum.PlatformStand = true end
			if not _flightBV or not _flightBV.Parent then
				_flightBV = Instance.new("BodyVelocity")
				_flightBV.MaxForce = Vector3.new(1e5,1e5,1e5)
				_flightBV.Name = "SosyFlightBV"
				_flightBV.Parent = hrp
			end
			local speed = (_G.FlightBoostActive and (_G.FlightBoostSpeed or 80)) or (_G.FlightBaseSpeed or 40)
			local cam   = workspace.CurrentCamera
			local cf    = cam.CFrame
			local dir   = Vector3.zero
			local uis   = game:GetService("UserInputService")
			if uis:IsKeyDown(Enum.KeyCode.W) then dir += cf.LookVector end
			if uis:IsKeyDown(Enum.KeyCode.S) then dir -= cf.LookVector end
			if uis:IsKeyDown(Enum.KeyCode.A) then dir -= cf.RightVector end
			if uis:IsKeyDown(Enum.KeyCode.D) then dir += cf.RightVector end
			if uis:IsKeyDown(Enum.KeyCode.Space) then dir += Vector3.yAxis end
			if uis:IsKeyDown(Enum.KeyCode.LeftControl) then dir -= Vector3.yAxis end
			_G.FlightBoostActive = uis:IsKeyDown(Enum.KeyCode.LeftShift)
			_flightBV.Velocity = dir.Magnitude > 0 and dir.Unit * speed or Vector3.zero
		end)
	end

	-- ── INVISIBLE ─────────────────────────────────────────────────────────────
	_G.SetInvisible = function(on)
		local char = getChar(); if not char then return end
		for _, p in ipairs(char:GetDescendants()) do
			if p:IsA("BasePart") and p.Name ~= "HumanoidRootPart" then
				p.LocalTransparencyModifier = on and 1 or 0
			end
		end
		if on then
			char.DescendantAdded:Connect(function(p)
				if _G.InvisibleEnabled and p:IsA("BasePart") and p.Name ~= "HumanoidRootPart" then
					p.LocalTransparencyModifier = 1
				end
			end)
		end
	end

	-- ── NOCLIP ────────────────────────────────────────────────────────────────
	local _noclipConn
	_G.SetNoclip = function(on)
		if _noclipConn then _noclipConn:Disconnect(); _noclipConn = nil end
		if not on then
			local char = getChar()
			if char then
				for _, p in ipairs(char:GetDescendants()) do
					if p:IsA("BasePart") then p.CanCollide = true end
				end
			end
			return
		end
		_noclipConn = RS.Stepped:Connect(function()
			if not _G.NoclipDomain then return end
			local char = getChar(); if not char then return end
			for _, p in ipairs(char:GetDescendants()) do
				if p:IsA("BasePart") then p.CanCollide = false end
			end
		end)
	end

	-- ── ESP ───────────────────────────────────────────────────────────────────
	local _espConns = {}
	local function clearESP()
		for _, c in ipairs(_espConns) do pcall(function() c:Disconnect() end) end
		_espConns = {}
		for _, pl in ipairs(Plrs:GetPlayers()) do
			if pl ~= me and pl.Character then
				local h = pl.Character:FindFirstChild("SosyESPHighlight")
				if h then h:Destroy() end
			end
		end
	end
	local function addESPToChar(char)
		if not char then return end
		if char:FindFirstChild("SosyESPHighlight") then return end
		local hl = Instance.new("Highlight")
		hl.Name = "SosyESPHighlight"
		hl.FillColor = Color3.fromRGB(255, 0, 0)
		hl.OutlineColor = Color3.fromRGB(255, 128, 0)
		hl.FillTransparency = 0.5
		hl.OutlineTransparency = 0
		hl.Adornee = char
		hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
		hl.Parent = char
	end
	_G.SetESP = function(on)
		clearESP()
		if not on then return end
		for _, pl in ipairs(Plrs:GetPlayers()) do
			if pl ~= me then
				if pl.Character then addESPToChar(pl.Character) end
				local c1 = pl.CharacterAdded:Connect(function(char) if _G.CharacterESP then addESPToChar(char) end end)
				local c2 = pl.CharacterRemoving:Connect(function(char)
					local h = char:FindFirstChild("SosyESPHighlight")
					if h then h:Destroy() end
				end)
				table.insert(_espConns, c1)
				table.insert(_espConns, c2)
			end
		end
		local c3 = Plrs.PlayerAdded:Connect(function(pl)
			if pl == me then return end
			local c1 = pl.CharacterAdded:Connect(function(char) if _G.CharacterESP then addESPToChar(char) end end)
			table.insert(_espConns, c1)
		end)
		table.insert(_espConns, c3)
	end

	-- ── NAME ESP ──────────────────────────────────────────────────────────────
	local _nameConns = {}
	local function clearNameESP()
		for _, c in ipairs(_nameConns) do pcall(function() c:Disconnect() end) end
		_nameConns = {}
		for _, pl in ipairs(Plrs:GetPlayers()) do
			if pl ~= me and pl.Character then
				local hrp = pl.Character:FindFirstChild("HumanoidRootPart")
				if hrp then
					local bb = hrp:FindFirstChild("SosyNameESP")
					if bb then bb:Destroy() end
				end
			end
		end
	end
	local function addNameESPToChar(pl, char)
		if not char then return end
		local hrp = char:FindFirstChild("HumanoidRootPart"); if not hrp then return end
		if hrp:FindFirstChild("SosyNameESP") then return end
		local bb = Instance.new("BillboardGui")
		bb.Name = "SosyNameESP"
		bb.Size = UDim2.fromOffset(120, 30)
		bb.StudsOffset = Vector3.new(0, 3, 0)
		bb.AlwaysOnTop = true
		bb.Parent = hrp
		local lbl = Instance.new("TextLabel")
		lbl.Size = UDim2.fromScale(1, 1)
		lbl.BackgroundTransparency = 1
		lbl.TextColor3 = Color3.fromRGB(255, 255, 80)
		lbl.TextStrokeTransparency = 0
		lbl.Font = Enum.Font.GothamBold
		lbl.TextScaled = true
		lbl.Text = pl.DisplayName
		lbl.Parent = bb
	end
	_G.SetNameESP = function(on)
		clearNameESP()
		if not on then return end
		for _, pl in ipairs(Plrs:GetPlayers()) do
			if pl ~= me then
				if pl.Character then addNameESPToChar(pl, pl.Character) end
				local c1 = pl.CharacterAdded:Connect(function(char) if _G.PlayerNameESP then addNameESPToChar(pl, char) end end)
				table.insert(_nameConns, c1)
			end
		end
		local c2 = Plrs.PlayerAdded:Connect(function(pl)
			if pl == me then return end
			local c1 = pl.CharacterAdded:Connect(function(char) if _G.PlayerNameESP then addNameESPToChar(pl, char) end end)
			table.insert(_nameConns, c1)
		end)
		table.insert(_nameConns, c2)
	end

	-- ── HITBOX EXPANDER ───────────────────────────────────────────────────────
	local _hitboxConn
	local _hitboxSaved = {}  -- [UserId] = original HRP Size

	local function restoreHitboxes(saved)
		pcall(function()
			for userId, origSize in pairs(saved) do
				local pl = Plrs:GetPlayerByUserId(userId)
				if pl and pl.Character then
					local hrp = pl.Character:FindFirstChild("HumanoidRootPart")
					if hrp then hrp.Size = origSize end
				end
			end
		end)
	end

	_G.SetHitboxExpander = function(on)
		if _hitboxConn then _hitboxConn:Disconnect(); _hitboxConn = nil end
		if not on then
			restoreHitboxes(_hitboxSaved)
			_hitboxSaved = {}
			return
		end
		local sz = _G.CharState and _G.CharState.HitboxSize or 8
		_hitboxConn = RS.Heartbeat:Connect(function()
			if not (_G.HitboxExpander and _G.CharState and _G.CharState.HitboxExpander) then return end
			for _, pl in ipairs(Plrs:GetPlayers()) do
				if pl ~= me and pl.Character then
					local hrp = pl.Character:FindFirstChild("HumanoidRootPart")
					if hrp then
						if not _hitboxSaved[pl.UserId] then
							_hitboxSaved[pl.UserId] = hrp.Size
						end
						local ts = math.clamp(_G.CharState.HitboxSize or sz, 1, 100)
						hrp.Size = Vector3.new(ts, ts, ts)
						hrp.Transparency = 1
						hrp.CanCollide = false
					end
				end
			end
		end)
	end

	-- ── HITBOX RANGE ──────────────────────────────────────────────────────────
	-- HitboxController.SphereHitbox is where this game decides who a move connects
	-- with: the client runs GetPartBoundsInRadius over workspace.Characters and then
	-- reports the survivors upward with FireServer(hits). Reach is computed here, not
	-- on the server, so scaling the radius at this single point extends every move
	-- that routes through it. Preferable to the HumanoidRootPart resize above — it
	-- covers NPCs and dummies too, and leaves other people's parts untouched.
	local _hbOrig, _hbCtrl = nil, nil
	_G.SetHitboxRange = function(on)
		pcall(function()
			local Knit = require(game:GetService("ReplicatedStorage").Knit.Knit)
			_hbCtrl = _hbCtrl or Knit.GetController("HitboxController")
			if not _hbCtrl then return end
			if not on then
				if _hbOrig then _hbCtrl.SphereHitbox = _hbOrig; _hbOrig = nil end
				return
			end
			if _hbOrig then return end
			_hbOrig = _hbCtrl.SphereHitbox
			local orig = _hbOrig
			_hbCtrl.SphereHitbox = function(self, char, cf, radius, ...)
				local r = radius
				if type(r) == "number" then
					local mult = tonumber(_G.CharState and _G.CharState.HitboxRange) or 1
					r = r * math.clamp(mult, 1, 12)
					-- Absolute ceiling on top of the multiplier. The hit list produced
					-- here is reported to the server, and on moves that close distance
					-- (grabs, chases) the server then moves you onto whoever you
					-- "reached" — so an unbounded radius yanks you across the map and
					-- reads as a fling. Capping the radius bounds how far you can be
					-- pulled, independently of the multiplier.
					local cap = tonumber(_G.CharState and _G.CharState.HitboxMaxRadius) or 30
					r = math.min(r, math.clamp(cap, 5, 120))
				end
				return orig(self, char, cf, r, ...)
			end
		end)
	end

	-- ── ANTI-STUN ─────────────────────────────────────────────────────────────
	local _antiStunHook
	_G.SetAntiStun = function(on)
		if not on then
			_G._AntiStunActive = false
			return
		end
		_G._AntiStunActive = true
		if _G._AntiStunHookSet then return end
		_G._AntiStunHookSet = true
		pcall(function()
			local blockWords = {"stun","knockback","ragdoll","flinch","hitstun"}
			local oldNc = hookmetamethod(game, "__namecall", function(self, ...)
				if not _G._AntiStunActive then return oldNc(self, ...) end
				local nm = getnamecallmethod and string.lower(getnamecallmethod()) or ""
				if nm == "fireserver" or nm == "invokeserver" then
					local args = {...}
					local first = tostring(args[1] or ""):lower()
					for _, w in ipairs(blockWords) do
						if first:find(w) then return end
					end
				end
				return oldNc(self, ...)
			end)
		end)
	end

	-- ── ANTI-RAGDOLL ──────────────────────────────────────────────────────────
	local _ragdollConn
	_G.SetAntiRagdoll = function(on)
		if _ragdollConn then _ragdollConn:Disconnect(); _ragdollConn = nil end
		if not on then return end
		local function disableRagdoll(char)
			if not char then return end
			for _, c in ipairs(char:GetDescendants()) do
				if c:IsA("BallSocketConstraint") or c:IsA("HingeConstraint") then
					c.Enabled = false
				end
			end
		end
		disableRagdoll(getChar())
		_ragdollConn = me.CharacterAdded:Connect(function(char)
			if _G.AntiRagdoll then disableRagdoll(char) end
		end)
	end

	-- ── AUTO BLOCK ────────────────────────────────────────────────────────────
	-- The real engine lives in the natives section (_G.SosyAutoBlock, ported from
	-- the xrezt auto block module). This is only a boot-order shim: applyNative can
	-- fire before the natives chunk has loaded, so we just record the desired state
	-- and hand off if the engine is already up. The engine re-reads
	-- _G.AutoBlockState on load, so nothing is lost either way.
	_G.SetAutoBlock = function(on)
		_G.AutoBlockState = _G.AutoBlockState or {}
		_G.AutoBlockState.Enabled = on == true
		if _G.SosyAutoBlock and type(_G.SosyAutoBlock.SetEnabled) == "function" then
			pcall(_G.SosyAutoBlock.SetEnabled, on == true)
		end
	end

	-- ── BLOCK BREAK ───────────────────────────────────────────────────────────
	local _blockBreakConn
	_G.SetBlockBreak = function(on)
		if _blockBreakConn then _blockBreakConn:Disconnect(); _blockBreakConn = nil end
		if not on then return end
		local bbRemote = nil
		local function findBBRemote()
			local remotes = workspace:FindFirstChild("Remotes") or game:GetService("ReplicatedStorage"):FindFirstChild("Remotes")
			if remotes then
				for _, r in ipairs(remotes:GetDescendants()) do
					if r:IsA("RemoteEvent") then
						local nl = r.Name:lower()
						if nl:find("break") or nl:find("blockbreak") then
							return r
						end
					end
				end
			end
		end
		local bbAcc = 0
		_blockBreakConn = RS.Heartbeat:Connect(function(dt)
			if not (_G.BlockBreakState and _G.BlockBreakState.Enabled) then return end
			bbAcc += dt
			if bbAcc < 0.08 then return end
			bbAcc = 0
			if not bbRemote or not bbRemote.Parent then
				bbRemote = findBBRemote()
			end
			if not bbRemote then return end
			pcall(function() bbRemote:FireServer() end)
		end)
	end
end
-- ─────────────────────────────────────────────────────────────────────────────

_G.SosyApplyMobileButtons = function(kind)
	kind = tostring(kind or "None")
	local pg = Players.LocalPlayer:FindFirstChild("PlayerGui") or Players.LocalPlayer:WaitForChild("PlayerGui")
	local old = pg:FindFirstChild("SosyMobileButtons")
	if old then old:Destroy() end
	if kind == "None" or kind == "" then return end
	local gui = Instance.new("ScreenGui")
	gui.Name = "SosyMobileButtons"
	gui.ResetOnSpawn = false
	gui.DisplayOrder = 260
	gui.Parent = pg
	local map = {
		Dash = { "Dash", function()
			_G.OriginalSideDashEnabled = true
			_G.DashAssistEnabled = true
			_G.DashAssistState = _G.DashAssistState or {}
			_G.DashAssistState.Enabled = true
			if type(_G.TriggerDashAssist) == "function" then pcall(_G.TriggerDashAssist) end
		end },
		Block = { "Block", function()
			_G.AutoBlockState = _G.AutoBlockState or {}
			local on = not (_G.AutoBlockState.Enabled == true)
			_G.AutoBlockState.Enabled = on
			if type(_G.SetAutoBlock) == "function" then pcall(_G.SetAutoBlock, on) end
		end },
		Lock = { "Lock", function()
			_G.LockState = _G.LockState or {}
			_G.LockState.Enabled = not (_G.LockState.Enabled == true)
		end },
		Blackflash = { "BF", function()
			local v = not (_G.NewBFCEnabled == true)
			_G.NewBFCEnabled = v
			_G.BFC1Enabled = v
			_G.BlackFlashState = _G.BlackFlashState or {}
			_G.BlackFlashState.Enabled = v
		end },
		Custom = { "Custom", function()
			if _G.SosySoftUI and _G.SosySoftUI.openAnimated then
				_G.SosySoftUI.openAnimated()
			elseif _G.SosySoftUIScreen then
				_G.SosySoftUIScreen.Enabled = true
			end
		end },
	}
	local def = map[kind] or map.Custom
	local b = Instance.new("TextButton")
	b.Size = UDim2.fromOffset(72, 72)
	b.Position = UDim2.new(1, -96, 1, -160)
	b.BackgroundColor3 = Color3.fromRGB(125, 60, 255)
	b.Text = def[1]
	b.TextColor3 = Color3.new(1, 1, 1)
	b.TextSize = 14
	b.Font = Enum.Font.GothamBold
	b.AutoButtonColor = true
	b.Parent = gui
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(1, 0)
	c.Parent = b
	b.MouseButton1Click:Connect(def[2])
end

local function httpRequest(opts)
	local req = (syn and syn.request) or http_request or request or (http and http.request)
	if req then
		return req(opts)
	end
	return game:GetService("HttpService"):RequestAsync(opts)
end

local function b64Basic(user, pass)
	local raw = tostring(user or "") .. ":" .. tostring(pass or "")
	local alphabet = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
	local function enc(data)
		local t = {}
		for i = 1, #data, 3 do
			local a, b, c = string.byte(data, i, i + 2)
			b, c = b or 0, c or 0
			local n = a * 65536 + b * 256 + c
			local n1 = math.floor(n / 262144) % 64
			local n2 = math.floor(n / 4096) % 64
			local n3 = math.floor(n / 64) % 64
			local n4 = n % 64
			local pad = (i + 2 > #data) and ((i + 1 > #data) and "==" or "=") or ""
			local chunk = alphabet:sub(n1 + 1, n1 + 1)
				.. alphabet:sub(n2 + 1, n2 + 1)
				.. (pad == "==" and "=" or alphabet:sub(n3 + 1, n3 + 1))
				.. (pad ~= "" and "=" or alphabet:sub(n4 + 1, n4 + 1))
			if pad == "==" then
				chunk = alphabet:sub(n1 + 1, n1 + 1) .. alphabet:sub(n2 + 1, n2 + 1) .. "=="
			elseif pad == "=" then
				chunk = alphabet:sub(n1 + 1, n1 + 1)
					.. alphabet:sub(n2 + 1, n2 + 1)
					.. alphabet:sub(n3 + 1, n3 + 1)
					.. "="
			end
			table.insert(t, chunk)
		end
		return table.concat(t)
	end
	return enc(raw)
end

_G.ApplyKillsSpoof = function(amount, on)
	amount = math.floor(tonumber(amount) or 0)
	on = on == true
	_G.KillsSpoofState = _G.KillsSpoofState or {}
	_G.KillsSpoofState.Enabled = on
	_G.KillsSpoofState.Amount = amount
	local lp = Players.LocalPlayer
	local name = tostring(_G.KillsSpoofState.Username or State["Kills Spoof Username"] or lp.Name)
	if name == "" then name = lp.Name end
	local pg = lp:FindFirstChild("PlayerGui") or lp:WaitForChild("PlayerGui")
	local gui = pg:FindFirstChild("SosyKillSpoofBoard")
	if not on then
		if gui then gui:Destroy() end
		return
	end
	if not gui then
		gui = Instance.new("ScreenGui")
		gui.Name = "SosyKillSpoofBoard"
		gui.ResetOnSpawn = false
		gui.DisplayOrder = 240
		gui.Parent = pg
		local frame = Instance.new("Frame")
		frame.Name = "Board"
		frame.Size = UDim2.fromOffset(168, 72)
		frame.Position = UDim2.fromOffset(16, 120)
		frame.BackgroundColor3 = Color3.fromRGB(28, 28, 32)
		frame.BorderSizePixel = 0
		frame.Active = true
		frame.Parent = gui
		local c = Instance.new("UICorner")
		c.CornerRadius = UDim.new(0, 8)
		c.Parent = frame
		do
			local uis = game:GetService("UserInputService")
			local dragging, start, startPos
			frame.InputBegan:Connect(function(inp)
				if inp.UserInputType == Enum.UserInputType.MouseButton1 or inp.UserInputType == Enum.UserInputType.Touch then
					dragging = true
					start = inp.Position
					startPos = frame.Position
					inp.Changed:Connect(function()
						if inp.UserInputState == Enum.UserInputState.End then dragging = false end
					end)
				end
			end)
			uis.InputChanged:Connect(function(inp)
				if dragging and (inp.UserInputType == Enum.UserInputType.MouseMovement or inp.UserInputType == Enum.UserInputType.Touch) then
					local d = inp.Position - start
					frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
				end
			end)
		end
		local x = Instance.new("TextButton")
		x.Size = UDim2.fromOffset(16, 16)
		x.Position = UDim2.fromOffset(4, 3)
		x.BackgroundTransparency = 1
		x.Text = "x"
		x.TextColor3 = Color3.fromRGB(220, 220, 220)
		x.TextSize = 12
		x.Font = Enum.Font.Gotham
		x.Parent = frame
		x.MouseButton1Click:Connect(function()
			_G.KillsSpoofState.Enabled = false
			gui:Destroy()
		end)
		local h1 = Instance.new("TextLabel")
		h1.Name = "HPeople"
		h1.BackgroundTransparency = 1
		h1.Size = UDim2.fromOffset(80, 14)
		h1.Position = UDim2.fromOffset(22, 4)
		h1.Font = Enum.Font.Gotham
		h1.TextSize = 11
		h1.TextColor3 = Color3.fromRGB(235, 235, 235)
		h1.TextXAlignment = Enum.TextXAlignment.Left
		h1.Text = "People"
		h1.Parent = frame
		local h2 = Instance.new("TextLabel")
		h2.Name = "HKills"
		h2.BackgroundTransparency = 1
		h2.Size = UDim2.fromOffset(60, 14)
		h2.Position = UDim2.new(1, -66, 0, 4)
		h2.Font = Enum.Font.Gotham
		h2.TextSize = 11
		h2.TextColor3 = Color3.fromRGB(235, 235, 235)
		h2.TextXAlignment = Enum.TextXAlignment.Right
		h2.Text = "Kills"
		h2.Parent = frame
		local n = Instance.new("TextLabel")
		n.Name = "NameLbl"
		n.BackgroundTransparency = 1
		n.Size = UDim2.new(1, -70, 0, 18)
		n.Position = UDim2.fromOffset(10, 28)
		n.Font = Enum.Font.GothamMedium
		n.TextSize = 13
		n.TextColor3 = Color3.fromRGB(245, 245, 245)
		n.TextXAlignment = Enum.TextXAlignment.Left
		n.TextTruncate = Enum.TextTruncate.AtEnd
		n.Parent = frame
		local k = Instance.new("TextLabel")
		k.Name = "KillsLbl"
		k.BackgroundTransparency = 1
		k.Size = UDim2.fromOffset(70, 18)
		k.Position = UDim2.new(1, -76, 0, 28)
		k.Font = Enum.Font.GothamMedium
		k.TextSize = 13
		k.TextColor3 = Color3.fromRGB(245, 245, 245)
		k.TextXAlignment = Enum.TextXAlignment.Right
		k.Parent = frame
	end
	local board = gui:FindFirstChild("Board")
	if board then
		local nl = board:FindFirstChild("NameLbl")
		local kl = board:FindFirstChild("KillsLbl")
		if nl then nl.Text = name end
		if kl then
			kl.Text = tostring(amount):reverse():gsub("(%d%d%d)", "%1,"):reverse():gsub("^,", "")
		end
	end
	-- also patch common leaderboard labels in PlayerGui
	pcall(function()
		for _, d in ipairs(pg:GetDescendants()) do
			if d:IsA("TextLabel") or d:IsA("TextButton") then
				local t = tostring(d.Text or "")
				if t == lp.Name or t == name then
					local sib = d.Parent and d.Parent:FindFirstChildWhichIsA("TextLabel")
					if sib and sib ~= d and tonumber((sib.Text or ""):gsub(",", "")) then
						sib.Text = tostring(amount)
					end
				end
			end
		end
	end)
end

_G.SosyAIAsk = function(userMsg, history)
	local HttpService = game:GetService("HttpService")
	local key = tostring(_G.SosyCursorApiKey or "")
	if key == "" and isfile and isfile("SosyHUB/cursor_key.txt") then
		key = tostring(readfile("SosyHUB/cursor_key.txt") or "")
	end
	-- Live model only — no canned tips / no unsolicited greetings
	local sys =
		"You are SosyHUB AI. Answer the user's message directly and briefly. Never invent tip lists, never start with a welcome/tutorial speech, no markdown fences."
	local lastErr = "no response"

	-- 1) VPS live AI proxy (Ollama / configured providers). Sends Cursor key for auth.
	local base = ""
	pcall(function()
		base = tostring((_G.SosyConfig and _G.SosyConfig.PublicBase) or _G.SosyPublicBase or "")
	end)
	if base ~= "" then
		local url = base:gsub("/$", "") .. "/public/sosy-ai"
		local ok, res = pcall(httpRequest, {
			Url = url,
			Method = "POST",
			Headers = {
				["Content-Type"] = "application/json",
				["Authorization"] = "Bearer " .. key,
			},
			Body = HttpService:JSONEncode({
				message = userMsg,
				history = history or {},
				key = key,
				system = sys,
				noTips = true,
			}),
		})
		if ok and type(res) == "table" then
			local code = tonumber(res.StatusCode or res.Status or 0) or 0
			local raw = res.Body or res.body or ""
			if code == 200 and type(raw) == "string" then
				local data = nil
				pcall(function() data = HttpService:JSONDecode(raw) end)
				if type(data) == "table" and type(data.reply) == "string" and data.reply ~= "" then
					return data.reply
				end
			elseif code == 402 then
				lastErr = "AI payment required (credits empty)"
			elseif code == 502 then
				lastErr = "VPS AI upstream busy — try again"
			elseif code > 0 then
				lastErr = "VPS AI HTTP " .. tostring(code)
			end
		end
	end

	-- 2) Direct gen.pollinations fallback (live model, not canned)
	do
		local allMsgs = {{ role = "system", content = sys }}
		for _, m in ipairs(history or {}) do
			table.insert(allMsgs, { role = (m.role == "assistant" and "assistant" or "user"), content = tostring(m.content) })
		end
		table.insert(allMsgs, { role = "user", content = tostring(userMsg) })
		local fOk, fRes = pcall(httpRequest, {
			Url = "https://gen.pollinations.ai/v1/chat/completions",
			Method = "POST",
			Headers = { ["Content-Type"] = "application/json", ["User-Agent"] = "SosyHUB/1.0" },
			Body = HttpService:JSONEncode({
				model = "openai-fast",
				messages = allMsgs,
			}),
		})
		if fOk and type(fRes) == "table" then
			local code = tonumber(fRes.StatusCode or fRes.Status or 0) or 0
			local raw = fRes.Body or fRes.body or ""
			if code == 200 and type(raw) == "string" and #raw > 0 then
				local ok2, data = pcall(HttpService.JSONDecode, HttpService, raw)
				if ok2 and type(data) == "table" and data.choices and data.choices[1] then
					local txt = (data.choices[1].message or {}).content or ""
					if #txt > 0 then return txt:sub(1, 1200) end
				end
			end
			if code > 0 then lastErr = "HTTP " .. tostring(code) end
		else
			lastErr = tostring(fRes)
		end
	end

	return nil, lastErr
end

-- Cursor API key (always refresh to the active key)
pcall(function()
	_G.SosyCursorApiKey = "crsr_8635ea9cc31aa22655bf9870418f6dba9ca078315b18d89e822fe69957fb5f29"
	ensureFolder()
	if writefile then
		writefile("SosyHUB/cursor_key.txt", tostring(_G.SosyCursorApiKey))
	end
end)

_G.SosyTeleportTo = _G.SosyTeleportTo or function(locName)
	locName = tostring(locName or "")
	local lp = Players.LocalPlayer
	local char = lp.Character
	local hrp = char and char:FindFirstChild("HumanoidRootPart")
	if not hrp then return false end

	-- Hardcoded positions from live workspace scan (Jujutsu Shenanigans)
	local LOCS = {
		["afk zone"]           = Vector3.new(172, 22, 201),
		["bowling alley"]      = Vector3.new(276, -70, -210),
		["cafeteria"]          = Vector3.new(-100, 56, -78),
		["city"]               = Vector3.new(-16, 23, 13),
		["domain"]             = Vector3.new(-74, -42, -118),
		["fight club"]         = Vector3.new(272, 84, 159),
		["gym"]                = Vector3.new(-138, 33, 33),
		["hospital"]           = Vector3.new(2, 43, -346),
		["jujutsu high"]       = Vector3.new(-100, 56, -78),
		["mall"]               = Vector3.new(139, -59, -250),
		["parking lot"]        = Vector3.new(272, 84, 155),
		["shibuya"]            = Vector3.new(172, 22, 14),
		["storage house"]      = Vector3.new(39, -18, -275),
		["towers"]             = Vector3.new(-94, 81, 114),
		["train button"]       = Vector3.new(186, -6, 564),
		["train station"]      = Vector3.new(162, 14, 553),
		["train station exit"] = Vector3.new(162, 14, 490),
		["tze's"]              = Vector3.new(-31, 41, 226),
		["under the map"]      = Vector3.new(-16, -150, 13),
		["unlicensed studios"] = Vector3.new(62, -41, -121),
		["grocery store"]      = Vector3.new(-249, 29, -109),
		["sewer"]              = Vector3.new(-74, -42, -118),
		["garage"]             = Vector3.new(272, 84, 159),
		["restaurant"]         = Vector3.new(-100, 56, -78),
		["courtroom"]          = Vector3.new(133, 35, -206),
	}

	local locKey = string.lower(locName)
	local target = LOCS[locKey]

	-- Fuzzy fallback: partial match
	if not target then
		for k, v in pairs(LOCS) do
			if k:find(locKey, 1, true) or locKey:find(k, 1, true) then
				target = v; break
			end
		end
	end

	-- Dynamic fallback: live workspace search
	if not target then
		local locStripped = locKey:gsub("%s+", "")
		local bestScore, bestPart = math.huge, nil
		pcall(function()
			for _, d in ipairs(workspace:GetDescendants()) do
				if d:IsA("BasePart") or d:IsA("Model") then
					local nl = d.Name:lower():gsub("%s+", "")
					local score
					if nl == locStripped then score = 0
					elseif nl:find(locStripped, 1, true) then score = 1
					elseif locStripped:find(nl, 1, true) and #nl > 3 then score = 2
					end
					if score and score < bestScore then
						bestScore = score
						if d:IsA("Model") then
							bestPart = d.PrimaryPart or d:FindFirstChildWhichIsA("BasePart")
						else
							bestPart = d
						end
					end
				end
			end
		end)
		if bestPart then target = bestPart.Position end
	end

	if target then
		hrp.CFrame = CFrame.new(target + Vector3.new(0, 4, 0))
		return true
	end
	return false
end

_G.ApplyAvatarSpoof = _G.ApplyAvatarSpoof or function(username)
	username = tostring(username or "")
	if username == "" then return end
	_G.AvatarSpoofUsername = username
	pcall(function()
		local userId = Players:GetUserIdFromNameAsync(username)
		local desc = Players:GetHumanoidDescriptionFromUserId(userId)
		local char = Players.LocalPlayer.Character
		local hum = char and char:FindFirstChildOfClass("Humanoid")
		if hum and desc then hum:ApplyDescription(desc) end
	end)
end

if not _G._SosyHitboxConn then
	local _sosyHbSaved  = {}   -- [UserId] = original Size
	local _sosyHbWasOn  = false

	_G._SosyHitboxConn = game:GetService("RunService").Stepped:Connect(function()
		local on = _G.HitboxExpander == true or (_G.CharState and _G.CharState.HitboxExpander)

		if not on then
			if _sosyHbWasOn then
				_sosyHbWasOn = false
				pcall(function()
					for userId, origSize in pairs(_sosyHbSaved) do
						local pl = Players:GetPlayerByUserId(userId)
						if pl and pl.Character then
							local hrp = pl.Character:FindFirstChild("HumanoidRootPart")
							if hrp then hrp.Size = origSize end
						end
					end
					_sosyHbSaved = {}
				end)
			end
			return
		end

		_sosyHbWasOn = true
		local size = math.clamp(tonumber(_G.HitboxSize) or (_G.CharState and _G.CharState.HitboxSize) or 5, 1, 100)
		local lp = Players.LocalPlayer
		pcall(function()
			for _, pl in ipairs(Players:GetPlayers()) do
				if pl == lp then continue end
				local char = pl.Character
				if not char then continue end
				local hrp = char:FindFirstChild("HumanoidRootPart")
				if hrp and hrp:IsA("BasePart") then
					if not _sosyHbSaved[pl.UserId] then
						_sosyHbSaved[pl.UserId] = hrp.Size
					end
					hrp.Size = Vector3.new(size, size, size)
					hrp.Transparency = 1
					hrp.CanCollide = false
				end
			end
		end)
	end)
end

_G.SosyBackend = Backend
return Backend

]====],
    ShaderPack = [====[-- Embedded pshade presets/skyboxes (no network required)
local EMBEDDED_PRESETS = {}
local EMBEDDED_SKYBOX = {}
local function _loadChunk(src)
	if type(src) ~= 'string' or src == '' then return nil end
	local fn = loadstring(src)
	if not fn then return nil end
	local ok, res = pcall(fn)
	if ok then return res end
	return nil
end
EMBEDDED_PRESETS['morning'] = _loadChunk([==[return {
		['fhnchvhfjsd']=-0.02,
		['ugtbbjhygt']=0.8,
		['tfbghuugbnjhg']=-0.5,
		['fvrtccvghghj']=Color3['fromRGB'](100,150,200),
		['jnfdhbnfcvh']=0.2,
		['fvtyghj']=5,
		['ygbhnj']=0.8,
		['njnfg']=2,
		['yfbghj']=Color3['fromRGB'](10,10,10),
		['tgvbyd']=7.5, 
		['ghuybhuyhj']=44, 
		['khnbfth']=1.5,
		['hgyghkg']=Color3['fromRGB'](0,0,0),
		['yfbhjku']=Color3['fromRGB'](200,200,200), 
		['ygyyfgvhbjytrt']=0.1, 
		['sdfcddc']=0.1, 
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](10,10,10),
		['hdfr7thgr']=0.3, 
		['gyhgtg']=0.6,
		['ygbhggv']=0.36,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		['shdbsnjfc']=0.2,
		['skdjfkdm']=0.5,
		['sjdjncdjf']=Color3['fromRGB'](70,120,170),
		['efjdjfk']=Color3['fromRGB'](10,50,100),
		['sejfd']=0.3,
		['jddfjsd']=1,
		['jdfkd']=0.5,
		['fvgsdfg']=15,
		['sdkvkflv']=5,
		['hbjhd']=0.5,
	}]==])
EMBEDDED_PRESETS['midday'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](242,243,243),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](2,2,2),
		['tgvbyd']=8, 
		['ghuybhuyhj']=-15.12, 
		['khnbfth']=3.25, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,247,237),
		['ygyyfgvhbjytrt']=0.203,
		['sdfcddc']=0.255, 
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](51,54,67),
		['hdfr7thgr']=0.85,
		['gyhgtg']=0.75,
		['ygbhggv']=0.26,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](175,221,255),
		['efjdjfk']=Color3['fromRGB'](13,105,172),
		['sejfd']=0.36,
		['jddfjsd']=0.72,
		['jdfkd']=0.277,
		['fvgsdfg']=21.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['afternoon'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.312,
		['fvrtccvghghj']=Color3['fromRGB'](242,243,243),
		['jnfdhbnfcvh']=0.31234,
		['fvtyghj']=10,
		['ygbhnj']=0.843,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](33,33,33),
		['tgvbyd']=14, 
		['ghuybhuyhj']=-15.12,
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,247,237),
		['ygyyfgvhbjytrt']=0.203,
		['sdfcddc']=0.255,
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](51,54,67),
		['hdfr7thgr']=0.85,
		['gyhgtg']=0.45,
		['ygbhggv']=0.23,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](175,221,255),
		['efjdjfk']=Color3['fromRGB'](13,105,172),
		['sejfd']=0.36,
		['jddfjsd']=0.72,
		['jdfkd']=0.217,
		['fvgsdfg']=21.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['evening'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](255, 205, 185),
		['jnfdhbnfcvh']=0.3234,
		['fvtyghj']=10,
		['ygbhnj']=0.813,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](2,2,2),
		['tgvbyd']=16, 
		['ghuybhuyhj']=45,
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0),
		['yfbhjku']=Color3['fromRGB'](255,247,237),
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.215, 
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.65, 
		['gyhgtg']=0.55,
		['ygbhggv']=0.43,
		['jghbjhgyfd']=Color3['fromRGB'](199,175,166),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=5.556,
		['sjdjncdjf']=Color3['fromRGB'](199,175,166),
		['efjdjfk']=Color3['fromRGB'](44,39,33),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=21.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['night'] = _loadChunk([==[return {
		['fhnchvhfjsd']=-0.06,
		['ugtbbjhygt']=-0.02,
		['tfbghuugbnjhg']=-0.2,
		['fvrtccvghghj']=Color3['fromRGB'](242,243,243),
		['jnfdhbnfcvh']=0.34,
		['fvtyghj']=10,
		['ygbhnj']=0.813,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](33,33,33),
		['tgvbyd']=20, 
		['ghuybhuyhj']=-15,
		['khnbfth']=3.25, 
		['hgyghkg']=Color3['fromRGB'](0,0,0),
		['yfbhjku']=Color3['fromRGB'](255,247,237),
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.255, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](51,54,67),
		['hdfr7thgr']=0.85, 
		['gyhgtg']=0.65,
		['ygbhggv']=0.33,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		['shdbsnjfc']=0.264,
		['skdjfkdm']=0.156,
		['sjdjncdjf']=Color3['fromRGB'](175,221,255),
		['efjdjfk']=Color3['fromRGB'](13,105,172),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['midnight'] = _loadChunk([==[return {
		['fhnchvhfjsd']=-0.1,
		['ugtbbjhygt']=-0.05,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](242,243,243),
		['jnfdhbnfcvh']=0.1,
		['fvtyghj']=10,
		['ygbhnj']=0.81,
		['njnfg']=6,
		['yfbghj']=Color3['fromRGB'](33,33,33),
		['tgvbyd']=0, 
		['ghuybhuyhj']=-0,
		['khnbfth']=2.25, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,247,237), 
		['ygyyfgvhbjytrt']=0.203,
		['sdfcddc']=0.255,
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](51,54,67),
		['hdfr7thgr']=0.15,
		['gyhgtg']=0.75,
		['ygbhggv']=0.33,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		['shdbsnjfc']=0.264,
		['skdjfkdm']=0.156,
		['sjdjncdjf']=Color3['fromRGB'](175,221,255),
		['efjdjfk']=Color3['fromRGB'](13,105,172),
		['sejfd']=0.16,
		['jddfjsd']=0.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['morninglite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=-0.02,
		['ugtbbjhygt']=0.4,
		['tfbghuugbnjhg']=-0.5,
		['fvrtccvghghj']=Color3['fromRGB'](100,150,200),
		['jnfdhbnfcvh']=0.135,
		['fvtyghj']=3,
		['ygbhnj']=0.452,
		['njnfg']=1.45654,
		['yfbghj']=Color3['fromRGB'](10,10,10),
		['tgvbyd']=7.5, 
		['ghuybhuyhj']=34, 
		['khnbfth']=1.5,
		['hgyghkg']=Color3['fromRGB'](0,0,0),
		['yfbhjku']=Color3['fromRGB'](200,200,200), 
		['ygyyfgvhbjytrt']=0.1, 
		['sdfcddc']=0.1, 
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](10,10,10),
		['hdfr7thgr']=0.3, 
		['gyhgtg']=0.134,
		['ygbhggv']=0.36,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		['shdbsnjfc']=0.2,
		['skdjfkdm']=0.5,
		['sjdjncdjf']=Color3['fromRGB'](70,120,170),
		['efjdjfk']=Color3['fromRGB'](10,50,100),
		['sejfd']=0.154,
		['jddfjsd']=0.3456,
		['jdfkd']=0.5,
		['fvgsdfg']=15,
		['sdkvkflv']=5,
		['hbjhd']=0.5,
	}]==])
EMBEDDED_PRESETS['middaylite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.15,
		['tfbghuugbnjhg']=-0.1543,
		['fvrtccvghghj']=Color3['fromRGB'](242,243,243),
		['jnfdhbnfcvh']=0.276,
		['fvtyghj']=7,
		['ygbhnj']=0.456,
		['njnfg']=2,
		['yfbghj']=Color3['fromRGB'](2,2,2),
		['tgvbyd']=8, 
		['ghuybhuyhj']=-15.12, 
		['khnbfth']=3.25, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,247,237),
		['ygyyfgvhbjytrt']=0.203,
		['sdfcddc']=0.255, 
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](51,54,67),
		['hdfr7thgr']=0.85,
		['gyhgtg']=0.75,
		['ygbhggv']=0.26,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		['shdbsnjfc']=0.264,
		['skdjfkdm']=0.156,
		['sjdjncdjf']=Color3['fromRGB'](175,221,255),
		['efjdjfk']=Color3['fromRGB'](13,105,172),
		['sejfd']=0.26,
		['jddfjsd']=0.72,
		['jdfkd']=0.277,
		['fvgsdfg']=2.54,
		['sdkvkflv']=3.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['afternoonlite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.25,
		['tfbghuugbnjhg']=-0.112,
		['fvrtccvghghj']=Color3['fromRGB'](242,243,243),
		['jnfdhbnfcvh']=0.1234,
		['fvtyghj']=5,
		['ygbhnj']=0.43,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](33,33,33),
		['tgvbyd']=14, 
		['ghuybhuyhj']=-15.12,
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,247,237),
		['ygyyfgvhbjytrt']=0.103,
		['sdfcddc']=0.255,
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](51,54,67),
		['hdfr7thgr']=0.85,
		['gyhgtg']=0.0145,
		['ygbhggv']=0.23,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		['shdbsnjfc']=0.164,
		['skdjfkdm']=0.256,
		['sjdjncdjf']=Color3['fromRGB'](175,221,255),
		['efjdjfk']=Color3['fromRGB'](13,105,172),
		['sejfd']=0.36,
		['jddfjsd']=0.72,
		['jdfkd']=0.217,
		['fvgsdfg']=21.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['eveninglite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.25,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](255,235,203),
		['jnfdhbnfcvh']=0.234,
		['fvtyghj']=3,
		['ygbhnj']=0.413,
		['njnfg']=4.34,
		['yfbghj']=Color3['fromRGB'](2,2,2),
		['tgvbyd']=16, 
		['ghuybhuyhj']=45,
		['khnbfth']=1.15,
		['hgyghkg']=Color3['fromRGB'](0,0,0),
		['yfbhjku']=Color3['fromRGB'](255,247,237),
		['ygyyfgvhbjytrt']=0.1203, 
		['sdfcddc']=0.1215, 
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.365, 
		['gyhgtg']=0.255,
		['ygbhggv']=0.143,
		['jghbjhgyfd']=Color3['fromRGB'](199,175,166),
		['shdbsnjfc']=0.164,
		['skdjfkdm']=5.356,
		['sjdjncdjf']=Color3['fromRGB'](199,175,166),
		['efjdjfk']=Color3['fromRGB'](44,39,33),
		['sejfd']=0.16,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=21.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['nightlite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=-0.06,
		['ugtbbjhygt']=-0.02,
		['tfbghuugbnjhg']=-0.2,
		['fvrtccvghghj']=Color3['fromRGB'](242,243,243),
		['jnfdhbnfcvh']=0.34,
		['fvtyghj']=4,
		['ygbhnj']=0.4513,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](33,33,33),
		['tgvbyd']=20, 
		['ghuybhuyhj']=-15,
		['khnbfth']=3.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0),
		['yfbhjku']=Color3['fromRGB'](255,247,237),
		['ygyyfgvhbjytrt']=0.103,
		['sdfcddc']=0.155,
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](51,54,67),
		['hdfr7thgr']=0.85,
		['gyhgtg']=0.15,
		['ygbhggv']=0.23,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		['shdbsnjfc']=0.264,
		['skdjfkdm']=0.156,
		['sjdjncdjf']=Color3['fromRGB'](175,221,255),
		['efjdjfk']=Color3['fromRGB'](13,105,172),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['midnightlite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=-0.1,
		['ugtbbjhygt']=-0.05,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](242,243,243),
		['jnfdhbnfcvh']=0.1,
		['fvtyghj']=4,
		['ygbhnj']=0.181,
		['njnfg']=3,
		['yfbghj']=Color3['fromRGB'](33,33,33), 
		['tgvbyd']=0, 
		['ghuybhuyhj']=-0,
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,247,237),
		['ygyyfgvhbjytrt']=0.1203,
		['sdfcddc']=0.1255, 
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](51,54,67),
		['hdfr7thgr']=0.05,
		['gyhgtg']=0.15,
		['ygbhggv']=0.23,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		['shdbsnjfc']=0.264,
		['skdjfkdm']=0.156,
		['sjdjncdjf']=Color3['fromRGB'](175,221,255),
		['efjdjfk']=Color3['fromRGB'](13,105,172),
		['sejfd']=0.16,
		['jddfjsd']=0.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['black'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](30,30,30),
		['jnfdhbnfcvh']=0.1,
		['fvtyghj']=10,
		['ygbhnj']=0.9,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](3,3,3),
		['tgvbyd']=5,
		['ghuybhuyhj']=-45,
		['khnbfth']=0.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0),
		['yfbhjku']=Color3['fromRGB'](255,255,255), 
		['ygyyfgvhbjytrt']=0.203,
		['sdfcddc']=0.255,
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.5,
		['gyhgtg']=0.45,
		['ygbhggv']=0.23,
		['jghbjhgyfd']=Color3['fromRGB'](0,0,0),
		['shdbsnjfc']=0.264,
		['skdjfkdm']=1.256,
		['sjdjncdjf']=Color3['fromRGB'](2,2,2),
		['efjdjfk']=Color3['fromRGB'](0,0,0),
		['sejfd']=0.16,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['green'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](132,182,141),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](2,2,2),
		['tgvbyd']=8, 
		['ghuybhuyhj']=-16, 
		['khnbfth']=4.25, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,255,255), 
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](75,151,75),
		['hdfr7thgr']=0.5, 
		['gyhgtg']=0.95,
		['ygbhggv']=0.63,
		['jghbjhgyfd']=Color3['fromRGB'](75,151,75),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](75,151,75),
		['efjdjfk']=Color3['fromRGB'](40,127,71),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['red'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](238,196,182),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](238,196,182), 
		['tgvbyd']=7, 
		['ghuybhuyhj']=-45, 
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,255,255), 
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25,
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](205,84,75),
		['hdfr7thgr']=0.5, 
		['gyhgtg']=0.95,
		['ygbhggv']=0.43,
		['jghbjhgyfd']=Color3['fromRGB'](217,133,108),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](205,84,75),
		['efjdjfk']=Color3['fromRGB'](217,133,108),
		['sejfd']=0.36,
		['jddfjsd']=6.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['yellow'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](247,241,141),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](247,241,141), 
		['tgvbyd']=7, 
		['ghuybhuyhj']=-45, 
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,255,255),
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25,
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](245,205,153),
		['hdfr7thgr']=0.5, 
		['gyhgtg']=0.95,
		['ygbhggv']=0.43,
		['jghbjhgyfd']=Color3['fromRGB'](249,233,153),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](245,205,153),
		['efjdjfk']=Color3['fromRGB'](249,233,153),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['pink'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](229,173,200),
		['jnfdhbnfcvh']=1,
		['fvtyghj']=58,
		['ygbhnj']=4,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](238,196,182),
		['tgvbyd']=7,
		['ghuybhuyhj']=-45, 
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0),
		['ygyyfgvhbjytrt']=0.203,
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.5, 
		['gyhgtg']=0.95,
		['ygbhggv']=0.63,
		['jghbjhgyfd']=Color3['fromRGB'](238,196,182),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](232,186,182),
		['efjdjfk']=Color3['fromRGB'](238,196,182),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['gray'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](199,193,183),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](2,2,2),
		['tgvbyd']=7, 
		['ghuybhuyhj']=-45,
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0),
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.5, 
		['gyhgtg']=0.95,
		['ygbhggv']=0.63,
		['jghbjhgyfd']=Color3['fromRGB'](109,110,108),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](161,165,162),
		['efjdjfk']=Color3['fromRGB'](109,110,108),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['white'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](242,243,243),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](2,2,2), 
		['tgvbyd']=10, 
		['ghuybhuyhj']=-45,
		['khnbfth']=2.25, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.203,
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](255,255,255),
		['hdfr7thgr']=0.5, 
		['gyhgtg']=0.95,
		['ygbhggv']=0.63,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](249,233,153),
		['efjdjfk']=Color3['fromRGB'](245,205,48),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['purple'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](165,165,203),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=46,
		['ygbhnj']=0.8,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](2,2,2), 
		['tgvbyd']=10, 
		['ghuybhuyhj']=-45,
		['khnbfth']=2.25, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0),
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.5,
		['gyhgtg']=0.65,
		['ygbhggv']=0.63,
		['jghbjhgyfd']=Color3['fromRGB'](104,116,172),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](107,50,124),
		['efjdjfk']=Color3['fromRGB'](104,116,172),
		['sejfd']=0.36,
		['jddfjsd']=7.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['blacklite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](30,30,30),
		['jnfdhbnfcvh']=0.1,
		['fvtyghj']=10,
		['ygbhnj']=0.9,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](35,35,35),
		['tgvbyd']=20,
		['ghuybhuyhj']=-45,
		['khnbfth']=0.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0),
		['yfbhjku']=Color3['fromRGB'](255,255,255), 
		['ygyyfgvhbjytrt']=0.203,
		['sdfcddc']=0.255,
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.5,
		['gyhgtg']=0.45,
		['ygbhggv']=0.23,
		['jghbjhgyfd']=Color3['fromRGB'](0,0,0),
		['shdbsnjfc']=0.264,
		['skdjfkdm']=1.256,
		['sjdjncdjf']=Color3['fromRGB'](2,2,2),
		['efjdjfk']=Color3['fromRGB'](0,0,0),
		['sejfd']=0.16,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['greenlite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](132,182,141),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=10,
		['ygbhnj']=0.3,
		['njnfg']=2,
		['yfbghj']=Color3['fromRGB'](2,2,2),
		['tgvbyd']=8, 
		['ghuybhuyhj']=-16, 
		['khnbfth']=4.25, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,255,255), 
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](75,151,75),
		['hdfr7thgr']=0.5, 
		['gyhgtg']=0.35,
		['ygbhggv']=0.23,
		['jghbjhgyfd']=Color3['fromRGB'](75,151,75),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.1556,
		['sjdjncdjf']=Color3['fromRGB'](75,151,75),
		['efjdjfk']=Color3['fromRGB'](40,127,71),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['redlite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](238,196,182),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=4,
		['ygbhnj']=0.8,
		['njnfg']=2,
		['yfbghj']=Color3['fromRGB'](238,196,182), 
		['tgvbyd']=7, 
		['ghuybhuyhj']=-45, 
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,255,255), 
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25,
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](205,84,75),
		['hdfr7thgr']=0.2, 
		['gyhgtg']=0.25,
		['ygbhggv']=0.13,
		['jghbjhgyfd']=Color3['fromRGB'](217,133,108),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](205,84,75),
		['efjdjfk']=Color3['fromRGB'](217,133,108),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['yellowlite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](247,241,141),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=5,
		['ygbhnj']=0.8,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](247,241,141), 
		['tgvbyd']=8, 
		['ghuybhuyhj']=-35, 
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](255,255,255),
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25,
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](245,205,153),
		['hdfr7thgr']=0.5, 
		['gyhgtg']=0.25,
		['ygbhggv']=0.23,
		['jghbjhgyfd']=Color3['fromRGB'](249,233,153),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](245,205,153),
		['efjdjfk']=Color3['fromRGB'](249,233,153),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['pinklite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](229,173,200),
		['jnfdhbnfcvh']=1,
		['fvtyghj']=8,
		['ygbhnj']=4,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](238,196,182),
		['tgvbyd']=7,
		['ghuybhuyhj']=-45, 
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0),
		['ygyyfgvhbjytrt']=0.203,
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true,
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.5, 
		['gyhgtg']=0.25,
		['ygbhggv']=0.13,
		['jghbjhgyfd']=Color3['fromRGB'](238,196,182),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](232,186,182),
		['efjdjfk']=Color3['fromRGB'](238,196,182),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['graylite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](199,193,183),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=6,
		['ygbhnj']=0.5,
		['njnfg']=5,
		['yfbghj']=Color3['fromRGB'](2,2,2),
		['tgvbyd']=7, 
		['ghuybhuyhj']=-35,
		['khnbfth']=2.25,
		['hgyghkg']=Color3['fromRGB'](0,0,0),
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.5, 
		['gyhgtg']=0.25,
		['ygbhggv']=0.13,
		['jghbjhgyfd']=Color3['fromRGB'](109,110,108),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](161,165,162),
		['efjdjfk']=Color3['fromRGB'](109,110,108),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.117,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['whitelite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](242,243,243),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=1,
		['ygbhnj']=0.8,
		
		['njnfg']=5,
		
		['yfbghj']=Color3['fromRGB'](2,2,2), 
		['tgvbyd']=10, 
		['ghuybhuyhj']=-45, 
		['khnbfth']=2.25, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](255,255,255),
		['hdfr7thgr']=0.5, 
		
		['gyhgtg']=0.15,
		['ygbhggv']=0.63,
		['jghbjhgyfd']=Color3['fromRGB'](255,255,255),
		
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](249,233,153),
		['efjdjfk']=Color3['fromRGB'](245,205,48),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['purplelite'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](165,165,203),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=6,
		['ygbhnj']=0.8,
		['njnfg']=2,
		['yfbghj']=Color3['fromRGB'](2,2,2),
		['tgvbyd']=8,
		['ghuybhuyhj']=-45,
		['khnbfth']=2.25, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.203,
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.5, 
		['gyhgtg']=0.35,
		['ygbhggv']=0.23,
		['jghbjhgyfd']=Color3['fromRGB'](104,116,172),
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](107,50,124),
		['efjdjfk']=Color3['fromRGB'](104,116,172),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['rain'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](200,200,200),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		
		['njnfg']=5,
		
		['yfbghj']=Color3['fromRGB'](109,110,108), 
		['tgvbyd']=15, 
		['ghuybhuyhj']=-45, 
		['khnbfth']=0.25, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.5, 
		
		['gyhgtg']=0.75,
		['ygbhggv']=0.93,
		['jghbjhgyfd']=Color3['fromRGB'](161,165,162),
		
		['shdbsnjfc']=0.564,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](161,165,162),
		['efjdjfk']=Color3['fromRGB'](161,165,162),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['snow'] = _loadChunk([==[return {
		['fhnchvhfjsd']=-0.1,
		['ugtbbjhygt']=0.2,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](200,200,255),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		
		['njnfg']=3,
		
		['yfbghj']=Color3['fromRGB'](242,243,243), 
		['tgvbyd']=10, 
		['ghuybhuyhj']=-45, 
		['khnbfth']=-0.1, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.403, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.2, 
		
		['gyhgtg']=0.55,
		['ygbhggv']=0.93,
		['jghbjhgyfd']=Color3['fromRGB'](242,243,243),
		
		['shdbsnjfc']=0.564,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](161,165,162),
		['efjdjfk']=Color3['fromRGB'](242,243,243),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['fog'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.2,
		['tfbghuugbnjhg']=0.3,
		['fvrtccvghghj']=Color3['fromRGB'](255,247,239),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		
		['njnfg']=3,
		
		['yfbghj']=Color3['fromRGB'](33,33,33), 
		['tgvbyd']=9, 
		['ghuybhuyhj']=-45, 
		['khnbfth']=-1.61, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.403, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](51,54,67),
		['hdfr7thgr']=0.72, 
		
		['gyhgtg']=0.75,
		['ygbhggv']=0.83,
		['jghbjhgyfd']=Color3['fromRGB'](242,243,243),
		
		['shdbsnjfc']=1.564,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](161,165,162),
		['efjdjfk']=Color3['fromRGB'](242,243,243),
		['sejfd']=0.36,
		['jddfjsd']=1.72,
		
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['sunny'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.4,
		['tfbghuugbnjhg']=-0.1,
		['fvrtccvghghj']=Color3['fromRGB'](255,247,239),
		['jnfdhbnfcvh']=0.5,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		
		['njnfg']=5,
		
		['yfbghj']=Color3['fromRGB'](255,255,255), 
		['tgvbyd']=12, 
		['ghuybhuyhj']=-1, 
		['khnbfth']=-5.61, 
		['hgyghkg']=Color3['fromRGB'](255,255,255), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.403, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](51,54,67),
		['hdfr7thgr']=0.72, 
		
		['gyhgtg']=0.55,
		['ygbhggv']=0.23,
		['jghbjhgyfd']=Color3['fromRGB'](242,243,243),
		
		['shdbsnjfc']=0.164,
		['skdjfkdm']=0.156,
		['sjdjncdjf']=Color3['fromRGB'](161,165,162),
		['efjdjfk']=Color3['fromRGB'](242,243,243),
		['sejfd']=0.36,
		['jddfjsd']=0.72,
		
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['cloudy'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0,
		['ugtbbjhygt']=-0.07,
		['tfbghuugbnjhg']=-0.1,
		['fvrtccvghghj']=Color3['fromRGB'](255,247,239),
		['jnfdhbnfcvh']=0.3,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		
		['njnfg']=5,
		
		['yfbghj']=Color3['fromRGB'](109,110,108), 
		['tgvbyd']=15, 
		['ghuybhuyhj']=34, 
		['khnbfth']=-0.61, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](0,0,0),
		['hdfr7thgr']=0.5, 
		
		['gyhgtg']=9.55,
		['ygbhggv']=0.93,
		['jghbjhgyfd']=Color3['fromRGB'](161, 165,162),
		
		['shdbsnjfc']=0.164,
		['skdjfkdm']=0.156,
		['sjdjncdjf']=Color3['fromRGB'](161,165,162),
		['efjdjfk']=Color3['fromRGB'](242,243,243),
		['sejfd']=0.36,
		['jddfjsd']=0.72,
		
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['storm'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](255,235,203),
		['jnfdhbnfcvh']=0.5,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		
		['njnfg']=7,
		
		['yfbghj']=Color3['fromRGB'](109,110,108), 
		['tgvbyd']=16, 
		['ghuybhuyhj']=45, 
		['khnbfth']=-0.11, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.403, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](51,54,67),
		['hdfr7thgr']=0.52, 
		
		['gyhgtg']=1.5,
		['ygbhggv']=0.93,
		['jghbjhgyfd']=Color3['fromRGB'](200,200,200),
		
		['shdbsnjfc']=0.564,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](161,165,162),
		['efjdjfk']=Color3['fromRGB'](161, 165,162),
		['sejfd']=0.36,
		['jddfjsd']=0.72,
		
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['autumn'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=-0.3,
		['fvrtccvghghj']=Color3['fromRGB'](255,165,0),
		['jnfdhbnfcvh']=0.5,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		
		['njnfg']=5,
		
		['yfbghj']=Color3['fromRGB'](255,140,0), 
		['tgvbyd']=8, 
		['ghuybhuyhj']=-15.12, 
		['khnbfth']=-2.61, 
		['hgyghkg']=Color3['fromRGB'](255,140,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.403, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](255,165,0),
		['hdfr7thgr']=0.72, 
		
		['gyhgtg']=0.55,
		['ygbhggv']=0.23,
		['jghbjhgyfd']=Color3['fromRGB'](242,243,243),
		
		['shdbsnjfc']=0.164,
		['skdjfkdm']=0.356,
		['sjdjncdjf']=Color3['fromRGB'](255,165,0),
		['efjdjfk']=Color3['fromRGB'](255,165,0),
		['sejfd']=0.36,
		['jddfjsd']=0.52,
		
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['spring'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.1,
		['ugtbbjhygt']=0.5,
		['tfbghuugbnjhg']=0.1,
		['fvrtccvghghj']=Color3['fromRGB'](255,247,239),
		['jnfdhbnfcvh']=0.5,
		['fvtyghj']=10,
		['ygbhnj']=0.8,
		
		['njnfg']=5,
		
		['yfbghj']=Color3['fromRGB'](173,216,230), 
		['tgvbyd']=9, 
		['ghuybhuyhj']=0, 
		['khnbfth']=-1.61, 
		['hgyghkg']=Color3['fromRGB'](0,0,0), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](144,238,144),
		['hdfr7thgr']=0.72, 
		
		['gyhgtg']=0.65,
		['ygbhggv']=0.53,
		['jghbjhgyfd']=Color3['fromRGB'](144,144,144),
		
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.156,
		['sjdjncdjf']=Color3['fromRGB'](173,216,230),
		['efjdjfk']=Color3['fromRGB'](144,238,144),
		['sejfd']=0.36,
		['jddfjsd']=0.72,
		
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['summer'] = _loadChunk([==[return {
		['fhnchvhfjsd']=0.15,
		['ugtbbjhygt']=0.6,
		['tfbghuugbnjhg']=0.1,
		['fvrtccvghghj']=Color3['fromRGB'](255,250,205),
		['jnfdhbnfcvh']=0.4,
		['fvtyghj']=12,
		['ygbhnj']=0.7,
		
		['njnfg']=4.5,
		
		['yfbghj']=Color3['fromRGB'](255,223,186), 
		['tgvbyd']=12, 
		['ghuybhuyhj']=1, 
		['khnbfth']=2.61, 
		['hgyghkg']=Color3['fromRGB'](255,223,186), 
		['yfbhjku']=Color3['fromRGB'](0,0,0), 
		['ygyyfgvhbjytrt']=0.403, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](255,223,186),
		['hdfr7thgr']=0.72, 
		
		['gyhgtg']=0.35,
		['ygbhggv']=0.83,
		['jghbjhgyfd']=Color3['fromRGB'](242,243,243),
		
		['shdbsnjfc']=0.464,
		['skdjfkdm']=0.656,
		['sjdjncdjf']=Color3['fromRGB'](255,223,186),
		['efjdjfk']=Color3['fromRGB'](255,200,160),
		['sejfd']=0.36,
		['jddfjsd']=0.72,
		
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_PRESETS['winter'] = _loadChunk([==[return {
		['fhnchvhfjsd']=-0.1,
		['ugtbbjhygt']=0.4,
		['tfbghuugbnjhg']=-0.2,
		['fvrtccvghghj']=Color3['fromRGB'](240,248,255),
		['jnfdhbnfcvh']=0.2,
		['fvtyghj']=8,
		['ygbhnj']=0.81,
		
		['njnfg']=6,
		
		['yfbghj']=Color3['fromRGB'](192,224,255), 
		['tgvbyd']=16, 
		['ghuybhuyhj']=45, 
		['khnbfth']=2.61, 
		['hgyghkg']=Color3['fromRGB'](245,245,245), 
		['yfbhjku']=Color3['fromRGB'](240,248,255), 
		['ygyyfgvhbjytrt']=0.203, 
		['sdfcddc']=0.25, 
		['hgnujuu7thgr']=true, 
		['hyhnngtf']=Color3['fromRGB'](192,224,255),
		['hdfr7thgr']=0.72, 
		
		['gyhgtg']=0.65,
		['ygbhggv']=0.93,
		['jghbjhgyfd']=Color3['fromRGB'](192,224,255),
		
		['shdbsnjfc']=0.364,
		['skdjfkdm']=0.556,
		['sjdjncdjf']=Color3['fromRGB'](192,224,255),
		['efjdjfk']=Color3['fromRGB'](200,200,200),
		['sejfd']=0.36,
		['jddfjsd']=0.72,
		
		['jdfkd']=0.217,
		['fvgsdfg']=11.54,
		['sdkvkflv']=16.77,
		['hbjhd']=0.277,
	}]==])
EMBEDDED_SKYBOX['morning'] = _loadChunk([==[return {
		['bk']='rbxassetid://7879352266',
		['dn']='rbxassetid://7879347786',
		['ft']='rbxassetid://7879358161',
		['lt']='rbxassetid://7879393853',
		['rt']='rbxassetid://7879350066',
		['up']='rbxassetid://7879345023'
	}]==])
EMBEDDED_SKYBOX['midday'] = _loadChunk([==[return {
		['bk']='rbxassetid://11832250366',
		['dn']='rbxassetid://11832250656',
		['ft']='rbxassetid://11832250915',
		['lt']='rbxassetid://11832250090',
		['rt']='rbxassetid://11832249798',
		['up']='rbxassetid://11832257515'
	}]==])
EMBEDDED_SKYBOX['afternoon'] = _loadChunk([==[return {
		['bk']='rbxassetid://225469345',
		['dn']='rbxassetid://225469349',
		['ft']='rbxassetid://225469359',
		['lt']='rbxassetid://225469364',
		['rt']='rbxassetid://225469372',
		['up']='rbxassetid://225469380'
	}]==])
EMBEDDED_SKYBOX['evening'] = _loadChunk([==[return {
		['bk']='rbxassetid://271042516',
		['dn']='rbxassetid://271077243',
		['ft']='rbxassetid://271042556',
		['lt']='rbxassetid://271042310',
		['rt']='rbxassetid://271042467',
		['up']='rbxassetid://271077958'
	}]==])
EMBEDDED_SKYBOX['rain'] = _loadChunk([==[return {
		['bk']='rbxassetid://4498828382',
		['dn']='rbxassetid://4498828812',
		['ft']='rbxassetid://4498829917',
		['lt']='rbxassetid://4498830911',
		['rt']='rbxassetid://4498830417',
		['up']='rbxassetid://4498831746'
	}]==])
EMBEDDED_SKYBOX['cloudy'] = _loadChunk([==[return {
		['bk']='rbxassetid://4495864450',
		['dn']='rbxassetid://4495864887',
		['ft']='rbxassetid://4495865458',
		['lt']='rbxassetid://4495866035',
		['rt']='rbxassetid://4495866584',
		['up']='rbxassetid://4495867486'
	}]==])
return { presets = EMBEDDED_PRESETS, skybox = EMBEDDED_SKYBOX }
]====],
    SosyShaders = [====[
--[[ SosyHUB Shaders — headless pshade ultimate engine (features only, SoftUI hosts controls) ]]
local HttpService = game:GetService("HttpService")
local Lighting = game:GetService("Lighting")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")

if _G._SosyShadersReady and _G.SosyShaders then
	return _G.SosyShaders
end

local lp = Players.LocalPlayer
local ws = workspace
local lg = Lighting
local terr = ws:FindFirstChildOfClass("Terrain") or ws.Terrain
local cam = ws.CurrentCamera
local pg = lp:WaitForChild("PlayerGui")

local fenv = getfenv and getfenv() or {}
local shp = fenv.sethiddenproperty or fenv.sethiddenprop or fenv.set_hidden_property or fenv.set_hidden_prop or function() end
local ghp = fenv.gethiddenproperty or fenv.gethiddenprop or function() end
local setclip = fenv.setclipboard or function() end
local readfile, writefile, isfile = fenv.readfile, fenv.writefile, fenv.isfile

local BASE = "https://raw.githubusercontent.com/randomstring0/pshade-ultimate/refs/heads/main/"
local SKY_BASE = BASE .. "sky/"
local PACK = (type(_G._SosyShaderPack) == "table" and _G._SosyShaderPack) or { presets = {}, skybox = {} }

local function httpgetLua(url)
	-- Prefer embedded pack (works offline / other PCs). Network is fallback only.
	local ok, res = pcall(function()
		return loadstring(game:HttpGet(url))()
	end)
	if ok then return res end
	return nil
end

local function packPreset(key)
	local t = PACK.presets
	if type(t) == "table" and type(t[key]) == "table" then return t[key] end
	return nil
end

local function packSky(key)
	local t = PACK.skybox
	if type(t) == "table" and type(t[key]) == "table" then return t[key] end
	return nil
end

local function randomName(l)
	l = l or 5
	local s = ""
	for _ = 1, l do
		local n = math.random(1, 3)
		if n == 1 then s = s .. string.char(math.random(65, 90))
		elseif n == 2 then s = s .. string.char(math.random(97, 122))
		else s = s .. tostring(math.random(0, 9)) end
	end
	return s
end

local technology = ghp(lg, "Technology") or "ShadowMap"
pcall(function() shp(lg, "Technology", "ShadowMap") end)

local colorcor, atmosphere, bloom, blur, depth, sky, sray, cloud
local restore, new = {}, {}
local originalPostEffectState = {}
local originalPostEffectProps = {}
local bestname = randomName()
local wshade = true
local check = true
local light
local shaderPresets = {}
local skybox = {}
local sett1 = nil
pcall(function() sett1 = settings() end)
if not sett1 then sett1 = { Rendering = { QualityLevel = Enum.QualityLevel.Automatic } } end

local function ensureEffects()
	local function take(className, enabled)
		local existing = lg:FindFirstChildOfClass(className)
		if existing then
			if enabled ~= nil and existing:IsA("PostEffect") then
				-- Keep the game's original post-effect state so Shaders OFF can restore it exactly.
				originalPostEffectState[existing] = existing.Enabled
				if className == "BlurEffect" then
					originalPostEffectProps[existing] = { Size = existing.Size }
				end
				existing.Enabled = false
			end
			if className == "BlurEffect" then existing.Size = 0 end
			local b = existing:Clone()
			b.Parent = lg
			if enabled ~= nil and b:IsA("PostEffect") then b.Enabled = true end
			b.Name = bestname
			table.insert(restore, b)
			table.insert(new, b)
			return b
		end
		return nil
	end

	bloom = take("BloomEffect", true) or bloom
	sky = take("Sky", nil) or sky
	atmosphere = take("Atmosphere", nil) or atmosphere
	blur = take("BlurEffect", true) or blur
	depth = take("DepthOfFieldEffect", true) or depth
	colorcor = take("ColorCorrectionEffect", true) or colorcor
	sray = take("SunRaysEffect", true) or sray

	if terr:FindFirstChildOfClass("Clouds") then
		cloud = terr:FindFirstChildOfClass("Clouds")
	end

	if not colorcor then
		colorcor = Instance.new("ColorCorrectionEffect")
		colorcor.Parent = lg
		colorcor.Name = bestname
		table.insert(new, colorcor)
	end
	if not atmosphere then
		atmosphere = Instance.new("Atmosphere")
		atmosphere.Parent = lg
		atmosphere.Density = 0
		atmosphere.Name = bestname
		table.insert(new, atmosphere)
	end
	if not bloom then
		bloom = Instance.new("BloomEffect")
		bloom.Parent = lg
		bloom.Name = bestname
		table.insert(new, bloom)
	end
	if not blur then
		blur = Instance.new("BlurEffect")
		blur.Parent = lg
		blur.Name = bestname
		table.insert(new, blur)
	end
	if not depth then
		depth = Instance.new("DepthOfFieldEffect")
		depth.Parent = lg
		depth.Name = bestname
		table.insert(new, depth)
	end
	if not sky then
		sky = Instance.new("Sky")
		sky.Parent = lg
		sky.Name = bestname
		table.insert(new, sky)
	end
	if not sray then
		sray = Instance.new("SunRaysEffect")
		sray.Parent = lg
		sray.Name = bestname
		table.insert(new, sray)
	end
	if not cloud then
		cloud = Instance.new("Clouds")
		cloud.Parent = terr
		cloud.Cover = 0
		cloud.Density = 0
		cloud.Name = bestname
		table.insert(new, cloud)
	end

	for _, v in ipairs(restore) do v.Name = bestname end
	for _, v in ipairs(new) do v.Name = bestname end

	lg.ChildRemoved:Connect(function(a)
		if a.Name ~= bestname then return end
		local v = a:Clone()
		v.Parent = lg
		v.Name = bestname
		if not v:IsA("Sky") and not v:IsA("Atmosphere") then table.insert(restore, v) end
		if v:IsA("BloomEffect") then bloom = v
		elseif v:IsA("Sky") then sky = v
		elseif v:IsA("Atmosphere") then atmosphere = v
		elseif v:IsA("BlurEffect") then blur = v
		elseif v:IsA("DepthOfFieldEffect") then depth = v
		elseif v:IsA("ColorCorrectionEffect") then colorcor = v
		elseif v:IsA("SunRaysEffect") then sray = v end
	end)

	terr.ChildRemoved:Connect(function(a)
		if a.Name ~= bestname then return end
		local v = a:Clone()
		v.Parent = terr
		v.Name = bestname
		if v:IsA("Clouds") then cloud = v end
	end)
end

ensureEffects()

local backup = {
	lighting = {
		ClockTime = lg.ClockTime,
		Ambient = lg.Ambient,
		Brightness = lg.Brightness,
		ColorShift_Bottom = lg.ColorShift_Bottom,
		ColorShift_Top = lg.ColorShift_Top,
		EnvironmentDiffuseScale = lg.EnvironmentDiffuseScale,
		EnvironmentSpecularScale = lg.EnvironmentSpecularScale,
		GlobalShadows = lg.GlobalShadows,
		OutdoorAmbient = lg.OutdoorAmbient,
		ShadowSoftness = lg.ShadowSoftness,
		technology = technology,
		GeographicLatitude = lg.GeographicLatitude,
		ExposureCompensation = lg.ExposureCompensation,
		FogEnd = lg.FogEnd,
		FogColor = lg.FogColor,
		FogStart = lg.FogStart,
	},
	terrain = {
		WaterColor = terr.WaterColor,
		WaterReflectance = terr.WaterReflectance,
		WaterTransparency = terr.WaterTransparency,
		WaterWaveSize = terr.WaterWaveSize,
		WaterWaveSpeed = terr.WaterWaveSpeed,
	},
}

local default = {
	yfbghj = lg.Ambient,
	tgvbyd = lg.ClockTime,
	ghuybhuyhj = lg.GeographicLatitude,
	khnbfth = lg.Brightness,
	hgyghkg = lg.ColorShift_Bottom,
	yfbhjku = lg.ColorShift_Top,
	ygyyfgvhbjytrt = lg.EnvironmentDiffuseScale,
	sdfcddc = lg.EnvironmentSpecularScale,
	hgnujuu7thgr = lg.GlobalShadows,
	hyhnngtf = lg.OutdoorAmbient,
	hdfr7thgr = lg.ExposureCompensation,
	fhnchvhfjsd = colorcor.Brightness,
	ugtbbjhygt = colorcor.Contrast,
	tfbghuugbnjhg = colorcor.Saturation,
	fvrtccvghghj = colorcor.TintColor,
	jnfdhbnfcvh = bloom.Intensity,
	fvtyghj = bloom.Size,
	ygbhnj = bloom.Threshold,
	njnfg = blur.Size,
	jdfkd = depth.FarIntensity,
	fvgsdfg = depth.FocusDistance,
	sdkvkflv = depth.InFocusRadius,
	hbjhd = depth.NearIntensity,
	gyhgtg = cloud.Cover,
	ygbhggv = cloud.Density,
	jghbjhgyfd = cloud.Color,
	shdbsnjfc = atmosphere.Density,
	skdjfkdm = atmosphere.Offset,
	sjdjncdjf = atmosphere.Color,
	efjdjfk = atmosphere.Decay,
	sejfd = atmosphere.Glare,
	jddfjsd = atmosphere.Haze,
}
light = default

local wl = {
	dof = true, cor = true, sray = true, bl = true, blr = false,
	rays = true, sflare = false, mblur = false, tech = "ShadowMap", global = false,
}
local adjust = { reflect = 0, waterspeed = nil }
local defsky = {
	bk = sky.SkyboxBk, dn = sky.SkyboxDn, ft = sky.SkyboxFt,
	lt = sky.SkyboxLf, rt = sky.SkyboxRt, up = sky.SkyboxUp,
}
local cussky = {
	bk = "rbxassetid://9544505500", dn = "rbxassetid://9544547905",
	ft = "rbxassetid://9544504852", lt = "rbxassetid://9544547694",
	rt = "rbxassetid://9544547542", up = "rbxassetid://9544547398",
}
skybox = { default = cussky, game = defsky }

do
	skybox.morning = packSky("morning") or httpgetLua(SKY_BASE .. "m.json") or cussky
	skybox.midday = packSky("midday") or httpgetLua(SKY_BASE .. "n.json") or cussky
	skybox.afternoon = packSky("afternoon") or httpgetLua(SKY_BASE .. "a.json") or cussky
	skybox.evening = packSky("evening") or httpgetLua(SKY_BASE .. "e.json") or cussky
	skybox.rain = packSky("rain") or httpgetLua(SKY_BASE .. "r.json") or cussky
	skybox.cloudy = packSky("cloudy") or httpgetLua(SKY_BASE .. "c.json") or cussky
end

-- FX-only ScreenGui (sunflare/motionblur), NOT a user interface
local fdist, fsize, ftrans = {1.7, 0.3, -0.3, -0.9}, {0.7, 0.2, 1.2, 0.45}, {0.8, 0.7, 0.9, 0.6}
local fls, rmod = {}, 1
local sre = Instance.new("ScreenGui")
sre.Name = "SosyShaderFX"
sre.ResetOnSpawn = false
sre.IgnoreGuiInset = true
sre.DisplayOrder = -1
sre.Enabled = false
pcall(function()
	sre.Parent = (gethui and gethui()) or game:GetService("CoreGui")
end)
if not sre.Parent then sre.Parent = pg end

local sflare = Instance.new("ImageLabel")
sflare.Name = "sflare"
sflare.BackgroundTransparency = 1
sflare.Image = "rbxassetid://15164863822"
sflare.ImageColor3 = Color3.new(1, 1, 0.8)
sflare.SizeConstraint = Enum.SizeConstraint.RelativeYY
sflare.ZIndex = 0
sflare.Size = UDim2.new(15 * 0.2, 0, 15 * 0.2, 0)
sflare.Parent = sre
sflare.Visible = false

task.spawn(function()
	for i = 1, #fdist do
		local f = Instance.new("ImageLabel")
		f.Name = "aflare"
		f.Size = UDim2.new(fsize[i] * 0.2, 0, fsize[i] * 0.2, 0)
		f.SizeConstraint = Enum.SizeConstraint.RelativeYY
		f.BackgroundTransparency = 1
		f.ImageTransparency = ftrans[i]
		f.BorderSizePixel = 0
		f.Rotation = -25
		f.Image = "rbxassetid://15164863822"
		f.ImageColor3 = Color3.new(1, 1, 0.8)
		f.ZIndex = -1
		f.Parent = sre
		f.Visible = false
		fls[#fls + 1] = f
	end
end)

local bmsize = 26
local mblur = Instance.new("BlurEffect")
mblur.Size = 0
mblur.Parent = cam
local bmut, ber, lor = 11, 5, cam.CFrame.LookVector
local flare, motionblur = false, false

ws:GetPropertyChangedSignal("CurrentCamera"):Connect(function()
	cam = ws.CurrentCamera
	if cam then
		mblur.Parent = cam
		if wl.mblur then mblur.Size = bmsize end
		lor = cam.CFrame.LookVector
	end
end)

local lightfolder = Instance.new("Folder")
lightfolder.Name = randomName()
lightfolder.Parent = ws

local globalil = {
	gridsize = 25, radius = 80,
	shadowdirection = {
		Vector3.new(1, -1, 0), Vector3.new(-1, -1, 0),
		Vector3.new(0, -1, 1), Vector3.new(0, -1, -1),
		Vector3.new(1, -1, 1), Vector3.new(-1, -1, -1),
		Vector3.new(1, -1, -1), Vector3.new(-1, -1, 1),
	},
	shadowlength = 8, reflectdis = 10, active = {},
}

local function roundToGrid(v)
	return Vector3.new(
		math.floor(v.X / globalil.gridsize + 0.5) * globalil.gridsize,
		math.floor(v.Y / globalil.gridsize + 0.5) * globalil.gridsize,
		math.floor(v.Z / globalil.gridsize + 0.5) * globalil.gridsize
	)
end
local function hashVec3(v) return tostring(v.X) .. "_" .. tostring(v.Y) .. "_" .. tostring(v.Z) end
local function isIndoors(pos)
	return ws:Raycast(pos, Vector3.new(0, 500, 0)) ~= nil
end
local function simulateShadow(pos)
	for _, dir in ipairs(globalil.shadowdirection) do
		if ws:Raycast(pos, dir.Unit * globalil.shadowlength) then return true end
	end
	return false
end
local function createLightPoint(pos)
	local part = Instance.new("Part")
	part.Anchored = true
	part.CanCollide = false
	part.Transparency = 1
	part.Size = Vector3.new(1, 1, 1)
	part.Position = pos
	part.Name = randomName()
	part.Parent = lightfolder
	local clight = Instance.new("PointLight")
	clight.Range = 24
	clight.Brightness = 0
	clight.Shadows = false
	clight.Color = light.yfbghj
	clight.Parent = part
	task.spawn(function()
		while part.Parent do
			local indoor = isIndoors(pos)
			local inShadow = simulateShadow(pos)
			local baseColor = indoor and Color3.fromRGB(255, 240, 220) or light.hyhnngtf
			local brightness = indoor and 0.35 or 0.75
			if inShadow then brightness = brightness * 0.35 end
			clight.Color = baseColor
			clight.Brightness = brightness
			task.wait(0.4)
		end
	end)
	return part
end

local function ob(u)
	return ws:FindPartOnRay(Ray.new(cam.CFrame.Position, lg:GetSunDirection() * 900), u) ~= nil
end
local function getsun()
	local camscreen = cam:WorldToScreenPoint(cam.CFrame.Position + lg:GetSunDirection())
	return Vector2.new(camscreen.X, camscreen.Y), camscreen.Z > 0
end
local function camcenter()
	return cam.ViewportSize / 2
end

local function toHex(v)
	if type(v) == "number" then
		return v % 1 == 0 and string.format("0x%X", v) or tostring(v)
	elseif type(v) == "string" then
		local h = {}
		for i = 1, #v do table.insert(h, string.format("\\x%02X", v:byte(i))) end
		return "\"" .. table.concat(h) .. "\""
	end
	return tostring(v)
end
local function serializeTable(t, i)
	i = i or 0
	local s = string.rep(" ", i)
	local r = "{\n"
	for k, v in pairs(t) do
		local kf = type(k) == "string" and "[\"" .. k .. "\"]" or "[" .. k .. "]"
		local vf
		if typeof(v) == "Color3" then
			vf = string.format("Color3.new(%g, %g, %g)", v.R, v.G, v.B)
		elseif type(v) == "table" then
			vf = serializeTable(v, i + 1)
		else
			vf = toHex(v)
		end
		r = r .. s .. " " .. kf .. "=" .. vf .. ",\n"
	end
	return r .. s .. "}"
end
local function deserializeTable(t)
	local function fix(x)
		for k, v in pairs(x) do
			if type(v) == "string" then
				if v:match("^0x[%x]+$") then x[k] = tonumber(v, 16)
				elseif v:match("\\x[%x][%x]") then
					x[k] = v:gsub("\\x(%x%x)", function(h) return string.char(tonumber(h, 16)) end)
				end
			elseif type(v) == "table" then fix(v) end
		end
		return x
	end
	return fix(t)
end

-- Load embedded presets first (offline / other PCs). Network fill-in is optional.
do
	local keys = {
		morning = "shr/morning.json", midday = "shr/midday.json", afternoon = "shr/afternoon.json",
		evening = "shr/evening%2Cjson", night = "shr/night.json", midnight = "shr/midnight.json",
		morninglite = "shr/morning1.json", middaylite = "shr/midday1.json", afternoonlite = "shr/afternoon1.json",
		eveninglite = "shr/evening1.json", nightlite = "shr/night1.json", midnightlite = "shr/midnight1.json",
		black = "shr/black.json", green = "shr/green.json", red = "shr/red.json", yellow = "shr/yellow.json",
		pink = "shr/pink.json", gray = "shr/gray.json", white = "shr/white.json", purple = "shr/purple.json",
		blacklite = "shr/black1.json", greenlite = "shr/green1.json", redlite = "shr/red1.json",
		yellowlite = "shr/yellow1.json", pinklite = "shr/pink1.json", graylite = "shr/gray1.json",
		whitelite = "shr/white1.json", purplelite = "shr/purple1.json",
		rain = "shr/rain.json", snow = "shr/snow.json", fog = "shr/fog.json", sunny = "shr/sunny.json",
		cloudy = "shr/cloudy.json", storm = "shr/storm.json",
		autumn = "shr/autumn.json", spring = "shr/spring.json", summer = "shr/summer.json", winter = "shr/winter.json",
	}
	for k, _ in pairs(keys) do
		local data = packPreset(k)
		if data then shaderPresets[k] = data end
	end
	_G._SosyShadersPresetsLoaded = true
	-- Optional network top-up for any missing keys (never required)
	task.spawn(function()
		for k, path in pairs(keys) do
			if not shaderPresets[k] then
				local data = httpgetLua(BASE .. path)
				if data then shaderPresets[k] = data end
			end
		end
	end)
end

if _G.saved then
	local ok, mf = pcall(function() return deserializeTable(_G.saved) end)
	if ok and type(mf) == "table" and mf.Shader then
		light = mf.Shader
		if mf.Skybox then cussky = mf.Skybox end
		if mf.Bloom then
			wl.bl = mf.Bloom[1]
			light.jnfdhbnfcvh = mf.Bloom[1]
			light.fvtyghj = mf.Bloom[2]
			light.ygbhnj = mf.Bloom[3]
		end
		if mf["Blur Effects"] ~= nil then wl.blr = mf["Blur Effects"] end
		if mf["Depth Of Field"] then
			depth.Enabled = mf["Depth Of Field"][1]
			light.jdfkd = mf["Depth Of Field"][2]
			light.fvgsdfg = mf["Depth Of Field"][3]
			light.sdkvkflv = mf["Depth Of Field"][4]
			light.hbjhd = mf["Depth Of Field"][5]
		end
		if mf.Atmosphere then
			light.shdbsnjfc = mf.Atmosphere[1]
			light.skdjfkdm = mf.Atmosphere[2]
			light.sejfd = mf.Atmosphere[3]
			light.jddfjsd = mf.Atmosphere[4]
		end
		if mf.Clouds then
			light.gyhgtg = mf.Clouds[1]
			light.ygbhggv = mf.Clouds[2]
		end
		if mf.ColorCorrection then
			wl.cor = mf.ColorCorrection[1]
			light.fhnchvhfjsd = mf.ColorCorrection[2]
			light.ugtbbjhygt = mf.ColorCorrection[3]
			light.tfbghuugbnjhg = mf.ColorCorrection[4]
		end
		if mf.Sunrays then
			wl.rays = mf.Sunrays[1]
			sray.Intensity = mf.Sunrays[2]
			sray.Spread = mf.Sunrays[3]
		end
		if mf.SunFlare ~= nil then wl.sflare = mf.SunFlare end
		if mf["Blur Motion"] ~= nil then wl.mblur = mf["Blur Motion"] end
	end
end

-- Camera FOV (persistent)
local cameraFov = 70
local cameraFovEnabled = true

local function enforceFov()
	if not cameraFovEnabled then return end
	local c = ws.CurrentCamera
	if not c then return end
	cam = c
	if math.abs(c.FieldOfView - cameraFov) > 0.05 then
		c.FieldOfView = cameraFov
	end
end

local function restoreWorld()
	sky.SkyboxBk = defsky.bk
	sky.SkyboxDn = defsky.dn
	sky.SkyboxFt = defsky.ft
	sky.SkyboxLf = defsky.lt
	sky.SkyboxRt = defsky.rt
	sky.SkyboxUp = defsky.up
	atmosphere.Density = default.shdbsnjfc
	atmosphere.Offset = default.skdjfkdm
	atmosphere.Color = default.sjdjncdjf
	atmosphere.Decay = default.efjdjfk
	atmosphere.Glare = default.sejfd
	atmosphere.Haze = default.jddfjsd
	cloud.Cover = default.gyhgtg
	cloud.Density = default.ygbhggv
	cloud.Color = default.jghbjhgyfd
	lg.Ambient = backup.lighting.Ambient
	lg.ClockTime = backup.lighting.ClockTime
	lg.GeographicLatitude = backup.lighting.GeographicLatitude
	lg.Brightness = backup.lighting.Brightness
	lg.ColorShift_Bottom = backup.lighting.ColorShift_Bottom
	lg.ColorShift_Top = backup.lighting.ColorShift_Top
	lg.EnvironmentDiffuseScale = backup.lighting.EnvironmentDiffuseScale
	lg.EnvironmentSpecularScale = backup.lighting.EnvironmentSpecularScale
	lg.GlobalShadows = backup.lighting.GlobalShadows
	lg.OutdoorAmbient = backup.lighting.OutdoorAmbient
	lg.ExposureCompensation = backup.lighting.ExposureCompensation
	lg.FogEnd = backup.lighting.FogEnd
	lg.FogColor = backup.lighting.FogColor
	lg.FogStart = backup.lighting.FogStart
	terr.WaterReflectance = backup.terrain.WaterReflectance
	terr.WaterTransparency = backup.terrain.WaterTransparency
	terr.WaterWaveSize = backup.terrain.WaterWaveSize
	terr.WaterWaveSpeed = backup.terrain.WaterWaveSpeed
	flare = false
	motionblur = false
	sre.Enabled = false
	mblur.Size = 0
	for hash, part in pairs(globalil.active) do
		if part and part:IsDescendantOf(ws) then part:Destroy() end
		globalil.active[hash] = nil
	end
end

local function applyShaderFrame()
	if not wshade then
		-- Shaders OFF = restore the game's original visual state and keep all
		-- Sosy-created post effects disabled. Do this every frame while off so
		-- nothing can immediately re-enable a shader effect behind the scenes.
		if check then check = false end
		restoreWorld()
		pcall(function() shp(lg, "Technology", technology) end)
		for obj, wasEnabled in pairs(originalPostEffectState) do
			if obj and obj.Parent == lg then
				obj.Enabled = wasEnabled == true
				local props = originalPostEffectProps[obj]
				if props and props.Size ~= nil and obj:IsA("BlurEffect") then
					obj.Size = props.Size
				end
			end
		end
		for _, v in ipairs(new) do
			if v and v.Parent == lg and v:IsA("PostEffect") then
				v.Enabled = false
			end
		end
		if mblur then mblur.Size = 0 end
		if sre then sre.Enabled = false end
		return
	end
	check = true

	if wl.global then
		task.spawn(function()
			local ql = sett1.Rendering and sett1.Rendering.QualityLevel
			if ql and ql ~= Enum.QualityLevel.Automatic then
				globalil.radius = (ql.Value < 5) and 80 or 200
			end
			local camPos = ws.CurrentCamera.CFrame.Position
			local min = camPos - Vector3.new(globalil.radius, globalil.radius, globalil.radius)
			local max = camPos + Vector3.new(globalil.radius, globalil.radius, globalil.radius)
			for x = min.X, max.X, globalil.gridsize do
				for y = min.Y, max.Y, globalil.gridsize do
					for z = min.Z, max.Z, globalil.gridsize do
						local pos = roundToGrid(Vector3.new(x, y, z))
						local hash = hashVec3(pos)
						if not globalil.active[hash] then
							if (pos - camPos).Magnitude <= globalil.radius then
								globalil.active[hash] = createLightPoint(pos)
							end
						end
					end
				end
			end
			for hash, part in pairs(globalil.active) do
				if part and part:IsDescendantOf(ws) then
					if (part.Position - camPos).Magnitude > (globalil.gridsize + globalil.radius) then
						part:Destroy()
						globalil.active[hash] = nil
					end
				end
			end
			task.wait(1.5)
		end)
	end

	if motionblur then
		local mag = (cam.CFrame.LookVector - lor).Magnitude
		mblur.Size = math.abs(mag) * bmut * ber / 2
		lor = cam.CFrame.LookVector
	else
		mblur.Size = 0
	end

	if flare then
		sre.Enabled = true
		local sunpos, front = getsun()
		local char = lp.Character
		local hrp = char and char:FindFirstChild("HumanoidRootPart")
		local clear = not ob((hrp and (hrp.Position + Vector3.new(0, 1.5, 0) - cam.CFrame.Position).Magnitude < 1.1) and char or nil)
		local tar = clear and 1 or 0
		rmod = rmod * (1 - 0.5) + tar * 0.5
		local scen = camcenter()
		local vec = sunpos - scen
		for i, v in ipairs(fls) do
			v.ImageTransparency = 1 - rmod + ftrans[i] * rmod
			local size, pos = v.AbsoluteSize, scen + vec * fdist[i]
			v.Position = UDim2.new(0.004, pos.X - size.X / 2, 0.004, pos.Y - size.Y / 2)
			v.Visible = front
		end
		sflare.ImageTransparency = 1 - rmod
		local zsize = sflare.AbsoluteSize
		sflare.Position = UDim2.new(0.004, sunpos.X - zsize.X / 2, 0.004, sunpos.Y - zsize.Y / 2)
		sflare.Visible = front
	else
		sre.Enabled = false
	end

	if light then
		flare = wl.sflare
		motionblur = wl.mblur
		sre.Enabled = wl.sflare
		sky.SkyboxBk = cussky.bk
		sky.SkyboxDn = cussky.dn
		sky.SkyboxFt = cussky.ft
		sky.SkyboxLf = cussky.lt
		sky.SkyboxRt = cussky.rt
		sky.SkyboxUp = cussky.up
		colorcor.Brightness = light.fhnchvhfjsd
		colorcor.Contrast = light.ugtbbjhygt
		colorcor.Saturation = light.tfbghuugbnjhg
		colorcor.TintColor = light.fvrtccvghghj
		colorcor.Enabled = wl.cor
		blur.Enabled = wl.blr
		bloom.Enabled = wl.bl
		bloom.Intensity = light.jnfdhbnfcvh
		bloom.Size = light.fvtyghj
		bloom.Threshold = light.ygbhnj
		blur.Size = light.njnfg
		lg.Ambient = light.yfbghj
		lg.ClockTime = light.tgvbyd
		lg.GeographicLatitude = light.ghuybhuyhj
		lg.Brightness = light.khnbfth
		lg.ColorShift_Bottom = light.hgyghkg
		lg.ColorShift_Top = light.yfbhjku
		lg.EnvironmentDiffuseScale = light.ygyyfgvhbjytrt
		lg.EnvironmentSpecularScale = light.sdfcddc
		lg.GlobalShadows = light.hgnujuu7thgr
		lg.OutdoorAmbient = light.hyhnngtf
		lg.ExposureCompensation = light.hdfr7thgr
		lg.FogEnd = math.huge
		lg.FogColor = Color3.fromRGB(255, 255, 255)
		lg.FogStart = math.huge
		cloud.Cover = light.gyhgtg
		cloud.Density = light.ygbhggv
		cloud.Color = light.jghbjhgyfd
		atmosphere.Density = light.shdbsnjfc
		atmosphere.Offset = light.skdjfkdm
		atmosphere.Color = light.sjdjncdjf
		atmosphere.Decay = light.efjdjfk
		atmosphere.Glare = light.sejfd
		atmosphere.Haze = light.jddfjsd
		depth.FarIntensity = light.jdfkd
		depth.FocusDistance = light.fvgsdfg
		depth.InFocusRadius = light.sdkvkflv
		depth.NearIntensity = light.hbjhd
		depth.Enabled = wl.dof
		sray.Enabled = wl.rays
	end
end

-- ONE centralized update loop (shaders + FOV)
if not _G._SosyCentralLoop then
	_G._SosyCentralLoop = RunService.PreRender:Connect(function()
		enforceFov()
		applyShaderFrame()
	end)
end

local function setEnabled(on)
	wshade = on == true
	if wshade then
		-- Disable the game's original post effects while the shader clone is active.
		for obj in pairs(originalPostEffectState) do
			if obj and obj.Parent == lg then obj.Enabled = false end
		end
	else
		-- Restore exactly what was enabled before SosyShaders took control.
		for obj, wasEnabled in pairs(originalPostEffectState) do
			if obj and obj.Parent == lg then obj.Enabled = wasEnabled == true end
		end
	end

	for _, v in pairs(ws:GetDescendants()) do
		if v:IsA("BasePart") then
			v.Reflectance = wshade and (adjust.reflect or 0) or 0
		end
	end

	pcall(function() shp(lg, "Technology", wshade and wl.tech or technology) end)

	for _, v in pairs(new) do
		if v and v.Parent == lg and v:IsA("PostEffect") then
			v.Enabled = wshade
		end
	end

	if not wshade then
		if sre then sre.Enabled = false end
		if mblur then mblur.Size = 0 end
		check = false
		restoreWorld()
		-- restoreWorld intentionally resets the world values; re-apply the original
		-- post-effect Enabled states after that reset.
		for obj, wasEnabled in pairs(originalPostEffectState) do
			if obj and obj.Parent == lg then obj.Enabled = wasEnabled == true end
		end
	else
		check = true
	end

	if not wl.global then
		for hash, part in pairs(globalil.active) do
			if part and part:IsDescendantOf(ws) then part:Destroy() end
			globalil.active[hash] = nil
		end
	end
end

local function setPreset(key)
	if key == "default" or key == nil then
		light = default
		return
	end
	local p = shaderPresets[key]
	if p then
		light = p
	end
end

local PRESET_BUTTONS = {
	["Default"] = "default",
	["Morning"] = "morning",
	["Midday"] = "midday",
	["Afternoon"] = "afternoon",
	["Evening"] = "evening",
	["Night"] = "night",
	["Midnight"] = "midnight",
	["Default Lite"] = "default",
	["Morning Lite"] = "morninglite",
	["Midday Lite"] = "middaylite",
	["Afternoon Lite"] = "afternoonlite",
	["Evening Lite"] = "eveninglite",
	["Night Lite"] = "nightlite",
	["Midnight Lite"] = "midnightlite",
	["Black Color"] = "black",
	["Green Color"] = "green",
	["Red Color"] = "red",
	["Yellow Color"] = "yellow",
	["Pink Color"] = "pink",
	["Gray Color"] = "gray",
	["White Color"] = "white",
	["Purple Color"] = "purple",
	["Black Color Lite"] = "blacklite",
	["Green Color Lite"] = "greenlite",
	["Red Color Lite"] = "redlite",
	["Yellow Color Lite"] = "yellowlite",
	["Pink Color Lite"] = "pinklite",
	["Gray Color Lite"] = "graylite",
	["White Color Lite"] = "whitelite",
	["Purple Color Lite"] = "purplelite",
	["Default Weather"] = "default",
	["Rain"] = "rain",
	["Snow"] = "snow",
	["Fog"] = "fog",
	["Sunny"] = "sunny",
	["Cloudy"] = "cloudy",
	["Storm"] = "storm",
	["Default Season"] = "default",
	["Autumn"] = "autumn",
	["Spring"] = "spring",
	["Summer"] = "summer",
	["Winter"] = "winter"
}

local Api = {}

function Api.setCameraFov(v)
	cameraFov = math.clamp(tonumber(v) or 70, 40, 120)
	enforceFov()
end
function Api.getCameraFov() return cameraFov end

function Api.applyControl(name, value)
	if name == "Shaders Enabled" then
		setEnabled(value == true)
	elseif name == "Camera FOV" then
		Api.setCameraFov(value)
	elseif name == "global illumination" then
		wl.global = value == true
		if not wl.global then
			for hash, part in pairs(globalil.active) do
				if part and part:IsDescendantOf(ws) then part:Destroy() end
				globalil.active[hash] = nil
			end
		end
	elseif name == "depthoffield enabled" then
		wl.dof = value == true
		depth.Enabled = value == true
	elseif name == "sunrays enabled" then
		wl.rays = value == true
		sray.Enabled = value == true
	elseif name == "colorcor enabled" then
		wl.cor = value == true
		colorcor.Enabled = value == true
	elseif name == "blureffect enabled" then
		wl.blr = value == true
		blur.Enabled = value == true
	elseif name == "bloom enabled" then
		wl.bl = value == true
		bloom.Enabled = value == true
	elseif name == "sunflare enabled" then
		wl.sflare = value == true
	elseif name == "blur motion enabled" then
		wl.mblur = value == true
	elseif name == "shaders technology" then
		wl.tech = tostring(value)
		if wshade then pcall(function() shp(lg, "Technology", wl.tech) end) end
	elseif name == "shaders quality" then
		local c = math.clamp(math.floor(tonumber(value) or 10), 1, 21)
		pcall(function()
			if c >= 10 then
				sett1.Rendering.QualityLevel = Enum.QualityLevel["Level" .. tostring(c)] or Enum.QualityLevel.Level10
			else
				sett1.Rendering.QualityLevel = Enum.QualityLevel["Level0" .. tostring(c)] or Enum.QualityLevel.Level01
			end
		end)
	elseif name == "skybox" then
		local a = tostring(value)
		cussky = skybox[a] or cussky
	elseif name == "reflectance" then
		adjust.reflect = tonumber(value) or 0
		for _, v in pairs(ws:GetDescendants()) do
			if v:IsA("BasePart") then
				v.Reflectance = wshade and adjust.reflect or 0
			end
		end
	elseif name == "waterwavespeed" then
		if wshade then terr.WaterWaveSpeed = tonumber(value) or terr.WaterWaveSpeed end
	elseif name == "watertransparency" then
		if wshade then terr.WaterTransparency = tonumber(value) or terr.WaterTransparency end
	elseif name == "waterwavesize" then
		if wshade then terr.WaterWaveSize = tonumber(value) or terr.WaterWaveSize end
	elseif name == "clock time" then
		light.tgvbyd = tonumber(value) or light.tgvbyd
	elseif name == "geographic latitude" then
		light.ghuybhuyhj = tonumber(value) or light.ghuybhuyhj
	elseif name == "clouds cover" then
		light.gyhgtg = tonumber(value) or light.gyhgtg
	elseif name == "clouds density" then
		light.ygbhggv = tonumber(value) or light.ygbhggv
	elseif name == "atmosphere density" then
		light.shdbsnjfc = tonumber(value) or light.shdbsnjfc
	elseif name == "atmosphere offset" then
		light.skdjfkdm = tonumber(value) or light.skdjfkdm
	elseif name == "atmosphere glare" then
		light.sejfd = tonumber(value) or light.sejfd
	elseif name == "atmosphere haze" then
		light.jddfjsd = tonumber(value) or light.jddfjsd
	elseif name == "dof farintensity" then
		light.jdfkd = tonumber(value) or light.jdfkd
	elseif name == "dof focusdistance" then
		light.fvgsdfg = tonumber(value) or light.fvgsdfg
	elseif name == "dof infocusradius" then
		light.sdkvkflv = tonumber(value) or light.sdkvkflv
	elseif name == "dof nearintensity" then
		light.hbjhd = tonumber(value) or light.hbjhd
	elseif name == "sunrays intensity" then
		sray.Intensity = tonumber(value) or sray.Intensity
	elseif name == "sunrays spread" then
		sray.Spread = tonumber(value) or sray.Spread
	elseif name == "colorcor brightness" then
		light.fhnchvhfjsd = tonumber(value) or light.fhnchvhfjsd
	elseif name == "colorcor contrast" then
		light.ugtbbjhygt = tonumber(value) or light.ugtbbjhygt
	elseif name == "colorcor saturation" then
		light.tfbghuugbnjhg = tonumber(value) or light.tfbghuugbnjhg
	elseif name == "blur size" then
		light.njnfg = tonumber(value) or light.njnfg
	elseif name == "bloom intensity" then
		light.jnfdhbnfcvh = tonumber(value) or light.jnfdhbnfcvh
	elseif name == "bloom size" then
		light.fvtyghj = tonumber(value) or light.fvtyghj
	elseif name == "bloom threshold" then
		light.ygbhnj = tonumber(value) or light.ygbhnj
	elseif name == "motion blur size" then
		bmsize = tonumber(value) or bmsize
	elseif name == "gui parent" or name == "language" or name == "background" or name == "Image background transparency" then
		-- Patrick UI chrome — SoftUI hosts controls; values stored only (no Patrick ScreenGui)
		if name == "language" and writefile then
			pcall(writefile, "pshade/lan", tostring(value))
		end
	end
end

function Api.click(name)
	if name == "copy saved adjustment to clipboard" then
		local a = {
			Skybox = cussky,
			Time = { light.tgvbyd, light.ghuybhuyhj },
			Clouds = { light.gyhgtg, light.ygbhggv },
			Atmosphere = { light.shdbsnjfc, light.skdjfkdm, light.sejfd, light.jddfjsd },
			["Depth Of Field"] = { depth.Enabled, light.jdfkd, light.fvgsdfg, light.sdkvkflv, light.hbjhd },
			Sunrays = { wl.rays, sray.Intensity, sray.Spread },
			ColorCorrection = { wl.cor, light.fhnchvhfjsd, light.ugtbbjhygt, light.tfbghuugbnjhg },
			["Blur Effects"] = wl.blr,
			Bloom = { wl.bl, light.jnfdhbnfcvh, light.fvtyghj, light.ygbhnj },
			SunFlare = wl.sflare,
			["Blur Motion"] = wl.mblur,
			Shader = light,
		}
		setclip("_G.saved = " .. serializeTable(a) .. " --tutorials : https://youtube.com/shorts/GJp-79AZ8I8?feature=share")
		return
	end
	local key = PRESET_BUTTONS[name]
	if key then setPreset(key) end
end

function Api.applyPlatformDefaults(mode)
	mode = tostring(mode or _G.SosyUIMode or "PC")
	if mode == "Mobile" then
		setEnabled(false)
		wl.rays = false
		if sray then sray.Enabled = false end
		return
	end
	-- PC: shaders on + Afternoon Lite + quality 21 + sunrays/colorcor + sat 1.00
	setEnabled(true)
	Api.setCameraFov(70)
	Api.click("Afternoon Lite")
	Api.applyControl("shaders quality", 21)
	Api.applyControl("sunrays enabled", true)
	Api.applyControl("sunrays intensity", 0.25)
	Api.applyControl("sunrays spread", 1)
	Api.applyControl("colorcor enabled", true)
	Api.applyControl("colorcor brightness", 0)
	Api.applyControl("colorcor contrast", 0)
	Api.applyControl("colorcor saturation", 1)
	Api.applyControl("bloom enabled", true)
	Api.applyControl("depthoffield enabled", true)
	Api.applyControl("skybox", "afternoon")
	pcall(function()
		if sray then sray.Enabled = true end
		if colorcor then colorcor.Enabled = true end
	end)
end

function Api.isShaderControl(name)
	if name == "Camera FOV" or name == "Shaders Enabled" then return true end
	if PRESET_BUTTONS[name] then return true end
	local known = {
		["gui parent"]=true, language=true, background=true, ["Image background transparency"]=true,
		["shaders technology"]=true, ["shaders quality"]=true, ["copy saved adjustment to clipboard"]=true,
		skybox=true, reflectance=true, ["global illumination"]=true,
		waterwavespeed=true, watertransparency=true, waterwavesize=true,
		["clock time"]=true, ["geographic latitude"]=true,
		["clouds cover"]=true, ["clouds density"]=true,
		["atmosphere density"]=true, ["atmosphere offset"]=true, ["atmosphere glare"]=true, ["atmosphere haze"]=true,
		["depthoffield enabled"]=true, ["dof farintensity"]=true, ["dof focusdistance"]=true,
		["dof infocusradius"]=true, ["dof nearintensity"]=true,
		["sunrays enabled"]=true, ["sunrays intensity"]=true, ["sunrays spread"]=true,
		["colorcor enabled"]=true, ["colorcor brightness"]=true, ["colorcor contrast"]=true, ["colorcor saturation"]=true,
		["blureffect enabled"]=true, ["blur size"]=true,
		["bloom enabled"]=true, ["bloom intensity"]=true, ["bloom size"]=true, ["bloom threshold"]=true,
		["sunflare enabled"]=true, ["blur motion enabled"]=true, ["motion blur size"]=true,
	}
	return known[name] == true
end

_G.SosyShaders = Api
_G._SosyShadersReady = true
_G.pshade = true -- prevent accidental original pshade double-load
return Api

]====],
    NovaCore = [====[
-- dumped by SosyHUB_LoadstringDump
-- time: 2026-07-21 20:12:09
-- tag: Core
-- bytes: 101055
-- SosyHUB Nova Full Natives (Ryu, BFC, Dash, Piano, Specials...)
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local RS = game:GetService("ReplicatedStorage")
local UIS = UserInputService
local lp = Players.LocalPlayer

local function ensure(tblName, defaults)
    _G[tblName] = _G[tblName] or {}
    if type(defaults) == "table" then
        for k, v in pairs(defaults) do
            if _G[tblName][k] == nil then _G[tblName][k] = v end
        end
    end
    return _G[tblName]
end

-- ===================== Ryu + BFC (from sosy hub) =====================
function cleanupRyuHeatUI(character)
    pcall(function()
        if not character then return end
        -- Light path only (full GetDescendants every frame was melting FPS)
        local function scrub(parent)
            if not parent then return end
            for _, child in ipairs(parent:GetChildren()) do
                local n = string.lower(child.Name)
                if child.Name == "Buildup" or string.find(n, "buildup") or string.find(n, "overheat") then
                    if child:IsA("BillboardGui") or child.Name == "Buildup" then
                        pcall(function() child:Destroy() end)
                    elseif child:IsA("ParticleEmitter") or child:IsA("Smoke") or child:IsA("Fire") then
                        pcall(function() child.Enabled = false end)
                    end
                end
            end
        end
        scrub(character:FindFirstChild("Torso") or character:FindFirstChild("UpperTorso"))
        scrub(character:FindFirstChild("Head"))
        scrub(character)
    end)
end

function isLocalRyuCharacter(character)
    return character ~= nil and character == lp.Character
end

-- Strip heat Buildup UI from everyone except local player (client-side only).
local lastForeignHeatStripAt = 0
function stripForeignRyuHeatUI()
    if not (_G.RyuState and _G.RyuState.LocalHeatBar) then return end
    local now = tick()
    if now - lastForeignHeatStripAt < 1.5 then return end
    lastForeignHeatStripAt = now
    pcall(function()
        local folder = workspace:FindFirstChild("Characters")
        if folder then
            for _, char in ipairs(folder:GetChildren()) do
                if not isLocalRyuCharacter(char) then
                    cleanupRyuHeatUI(char)
                end
            end
        end
    end)
end

local ryuHeatLoopConn = nil
local ryuSilentVentLoopConn = nil
local ryuRecoveryAnimConns = {}
local ryuAttrConns = {}
local ryuHeatValueHooks = {}
local lastRyuVentAt = 0
local ryuUnlockUntil = 0
local ryuSprintTrack = nil
local RYU_R_PHASE_DURATION = 1.35 -- short unlock window; recovery is fast-forwarded
local RYU_RECOVERY_SPEED = 12 -- AdjustSpeed on Recovery tracks (finish almost instantly)

-- Animations.Ryu.Recovery / Recovery2 / Recovery3
local RYU_RECOVERY_ANIM_IDS = {
    ["135300554211825"] = true,
    ["135915491872296"] = true,
    ["91450617822876"] = true,
}

-- Sounds.Ryu.Recovery / Recovery2 / Recovery3
local RYU_RECOVERY_SOUND_IDS = {
    ["120486776927289"] = true, -- Recovery3
    ["77659919167587"] = true,  -- Recovery.Sweet
    ["77417792700571"] = true,  -- Recovery2.Comb
    ["82086397349945"] = true,  -- Recovery.Comb
    ["140476433079342"] = true, -- Recovery2.Hair
}

function isRyuRecoverySoundId(soundId)
    local num = string.match(tostring(soundId or ""), "%d+")
    return num and RYU_RECOVERY_SOUND_IDS[num] == true
end

-- Client-local only: mute YOUR Recovery SFX. Other players still hear your R on their clients.
function silenceRyuRecoverySounds(char)
    char = char or lp.Character
    if not char then return end
    local function kill(snd)
        if not snd or not snd:IsA("Sound") then return end
        -- Only sounds parented under local character (not other players / global FX)
        if not snd:IsDescendantOf(char) then return end
        if not isRyuRecoverySoundId(snd.SoundId) then
            local n = string.lower(tostring(snd.Name or ""))
            if not string.find(n, "recover") then return end
        end
        pcall(function()
            snd.Volume = 0
            snd:Stop()
            snd:Destroy()
        end)
    end
    pcall(function()
        for _, d in ipairs(char:GetDescendants()) do
            if d:IsA("Sound") then kill(d) end
        end
    end)
end

-- General run (Vessel / default): StarterPlayer.StarterCharacterScripts.Animate.walk.WalkAnim
-- (Misc.Movement.Sprint is the Gojo-looking sprint — not used here)
local RYU_SPRINT_ANIM_ID = "rbxassetid://96489184596023"

function disconnectRyuAttrConns()
    for _, c in ipairs(ryuAttrConns) do
        pcall(function() c:Disconnect() end)
    end
    ryuAttrConns = {}
    for _, c in ipairs(ryuHeatValueHooks) do
        pcall(function() c:Disconnect() end)
    end
    ryuHeatValueHooks = {}
end

function disconnectRyuRecoveryAnimConns()
    for _, c in ipairs(ryuRecoveryAnimConns) do
        pcall(function() c:Disconnect() end)
    end
    ryuRecoveryAnimConns = {}
end

function isRyuRecoveryAnimId(animId)
    local num = string.match(tostring(animId or ""), "%d+")
    return num and RYU_RECOVERY_ANIM_IDS[num] == true
end

function isRyuMoveset()
    local char = lp.Character
    local ms = (char and char:GetAttribute("Moveset")) or lp:GetAttribute("Moveset")
    return ms == "Ryu"
end

function isRyuRPhase()
    return tick() < ryuUnlockUntil
end

function beginRyuRPhase(duration)
    ryuUnlockUntil = tick() + (duration or RYU_R_PHASE_DURATION)
end

function stopRyuSprintTrack()
    if ryuSprintTrack then
        pcall(function() ryuSprintTrack:Stop(0.15) end)
        ryuSprintTrack = nil
    end
end

function getRyuSprintAnim(char)
    local anim = nil
    -- Prefer live character Animate (same as Vessel / default run)
    pcall(function()
        local an = char and char:FindFirstChild("Animate")
        local walk = an and an:FindFirstChild("walk")
        anim = walk and (walk:FindFirstChild("WalkAnim") or walk:FindFirstChildOfClass("Animation"))
    end)
    if not (anim and anim:IsA("Animation")) then
        pcall(function()
            anim = game:GetService("StarterPlayer").StarterCharacterScripts.Animate.walk.WalkAnim
        end)
    end
    if anim and anim:IsA("Animation") then
        return anim
    end
    local fallback = Instance.new("Animation")
    fallback.AnimationId = RYU_SPRINT_ANIM_ID
    return fallback
end

function updateRyuSprintDuringR(char)
    if not isRyuRPhase() then
        stopRyuSprintTrack()
        return
    end
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not hum then
        stopRyuSprintTrack()
        return
    end
    local moving = hum.MoveDirection.Magnitude > 0.08
        and hum:GetState() ~= Enum.HumanoidStateType.Freefall
        and hum:GetState() ~= Enum.HumanoidStateType.Jumping
    local animator = hum:FindFirstChildOfClass("Animator")
    if not animator then
        stopRyuSprintTrack()
        return
    end
    if moving then
        if not ryuSprintTrack or not ryuSprintTrack.IsPlaying then
            pcall(function()
                ryuSprintTrack = animator:LoadAnimation(getRyuSprintAnim(char))
                ryuSprintTrack.Priority = Enum.AnimationPriority.Movement
                ryuSprintTrack.Looped = true
                ryuSprintTrack:Play(0.12)
                -- Match game sprint feel (WalkSpeed * 1.375)
                ryuSprintTrack:AdjustSpeed(1.35)
            end)
        end
    else
        stopRyuSprintTrack()
    end
end

-- Orange bar = Info.Overheated (BoolValue). Server rejects Granite until R/RightActivated clears it.
function isRyuOrange(char)
    char = char or lp.Character
    local info = char and char:FindFirstChild("Info")
    if not info then return false end
    local ohed = info:FindFirstChild("Overheated")
    if ohed and ohed:IsA("BoolValue") and ohed.Value == true then
        return true
    end
    local oh = info:FindFirstChild("Overheat")
    if oh and (oh:IsA("NumberValue") or oh:IsA("IntValue")) and oh.Value >= 48 then
        return true
    end
    return false
end

function forceClearClientHeatFlags(char)
    char = char or lp.Character
    local info = char and char:FindFirstChild("Info")
    if not info then return end
    local ohed = info:FindFirstChild("Overheated")
    if ohed and ohed:IsA("BoolValue") and ohed.Value then
        pcall(function() ohed.Value = false end)
    end
    -- Silent Vent must not zero Overheat: the stock RyuController.Overheat UI
    -- reads this value to size the Buildup bar. Only explicit HideHeatBar mode
    -- is allowed to pin the local meter to zero.
    if _G.RyuState and _G.RyuState.HideHeatBar and not _G.RyuState.LocalHeatBar then
        local oh = info:FindFirstChild("Overheat")
        if oh and (oh:IsA("NumberValue") or oh:IsA("IntValue")) and oh.Value ~= 0 then
            pcall(function() oh.Value = 0 end)
        end
    end
end

-- Fast-forward Recovery anim (speed + skip to end) then strip stun.
function fastForwardRyuRecoveryTracks(char)
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    local animator = hum and hum:FindFirstChildOfClass("Animator")
    if not animator then return end
    for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
        local aid = track.Animation and track.Animation.AnimationId
        local name = string.lower(tostring(track.Name or ""))
        if isRyuRecoveryAnimId(aid) or string.find(name, "recover") then
            pcall(function()
                track:AdjustSpeed(RYU_RECOVERY_SPEED)
                local len = track.Length
                if typeof(len) == "number" and len > 0.05 then
                    track.TimePosition = math.max(0, len - 0.03)
                end
                if _G.RyuState and _G.RyuState.SilentVent then
                    track:Stop(0)
                end
            end)
        end
    end
end

function cancelRyuRecovery(char)
    char = char or lp.Character
    if not char then return end
    if not isRyuRPhase() then return end
    pcall(function()
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            hum.AutoRotate = true
            if hum.WalkSpeed < 20 then hum.WalkSpeed = 20 end
            if hum.UseJumpPower ~= false then
                if (hum.JumpPower or 0) < 50 then hum.JumpPower = 50 end
            elseif (hum.JumpHeight or 0) < 7 then
                hum.JumpHeight = 7.2
            end
            fastForwardRyuRecoveryTracks(char)
        end
        silenceRyuRecoverySounds(char)
        forceClearClientHeatFlags(char)
        local info = char:FindFirstChild("Info")
        if info then
            for _, child in ipairs(info:GetChildren()) do
                if child.Name == "Stun" or child.Name == "Knockback" or child.Name == "NoSprint" then
                    pcall(function() child:Destroy() end)
                end
            end
        end
        for _, child in ipairs(char:GetChildren()) do
            if child.Name == "Stun" or child.Name == "Knockback" then
                pcall(function() child:Destroy() end)
            end
        end
        pcall(function()
            if char:GetAttribute("Stun") ~= nil then char:SetAttribute("Stun", 0) end
        end)
    end)
end

-- Vent to clear SERVER Overheated, strip Info.Stun, then Granite can land.
function ventClearOrangeForGranite()
    local char = lp.Character
    if not char then return end
    beginRyuRPhase(2.2)
    lastRyuVentAt = tick()
    pcall(function()
        RS.Knit.Knit.Services.RyuService.RE.RightActivated:FireServer()
    end)
    pcall(function()
        local Knit = require(RS.Knit.Knit)
        local tc = Knit.GetController("ToolController")
        if tc and type(tc.UseSetService) == "function" then
            local target = nil
            pcall(function() target = tc:GetTarget() end)
            tc:UseSetService("RightActivated", target)
        end
    end)
    forceClearClientHeatFlags(char)
    cancelRyuRecovery(char)
    -- Keep clearing while server applies vent stun / Overheated flip
    task.spawn(function()
        for _ = 1, 25 do
            if not isRyuRPhase() then break end
            forceClearClientHeatFlags(char)
            cancelRyuRecovery(char)
            task.wait(0.03)
        end
    end)
end

-- Every client-reachable Granite Blast fire path (server still validates Overheated).
local lastGraniteFireAt = 0
local ryuGraniteBusy = false
function fireGraniteBlastAllPaths(opts)
    opts = opts or {}
    if not isRyuMoveset() then return end
    if ryuGraniteBusy and not opts.fromUnlock then return end
    local now = tick()
    if now - lastGraniteFireAt < 0.15 then return end
    lastGraniteFireAt = now
    local char = lp.Character
    if not char then return end

    -- Turuncu iken önce R vent (server Overheated clear) — yoksa 1 yemez
    local orange = isRyuOrange(char)
    if orange and not opts.alreadyVented then
        ryuGraniteBusy = true
        ventClearOrangeForGranite()
        -- Wait until orange flag drops (or timeout) — server rejects Granite while Overheated
        for _ = 1, 20 do
            forceClearClientHeatFlags(char)
            cancelRyuRecovery(char)
            if not isRyuOrange(char) then break end
            task.wait(0.03)
        end
        task.wait(0.05)
        forceClearClientHeatFlags(char)
        cancelRyuRecovery(char)
        lastGraniteFireAt = 0
        fireGraniteBlastAllPaths({
            alreadyVented = true,
            fromUnlock = true,
            skipToolController = opts.skipToolController,
        })
        ryuGraniteBusy = false
        return
    end

    if isRyuRPhase() or orange then
        forceClearClientHeatFlags(char)
        cancelRyuRecovery(char)
    end

    local tool = char:FindFirstChild("Moveset") and char.Moveset:FindFirstChild("Granite Blast")
    if not tool then return end
    local air = false
    pcall(function()
        local hum = char:FindFirstChildOfClass("Humanoid")
        air = hum and hum.FloorMaterial == Enum.Material.Air or false
        if not air and char.PrimaryPart then
            local hit = workspace:Raycast(char.PrimaryPart.Position, Vector3.new(0, -4, 0), _G.MapParams)
            air = hit == nil
        end
    end)
    -- HOLD skill: heat scales with press duration on server.
    -- Always send Activated then near-instant Deactivated (tap / low charge).
    local function fireGraniteTap(skipTC)
        if not skipTC then
            pcall(function()
                local Knit = require(RS.Knit.Knit)
                local tc = Knit.GetController("ToolController")
                if tc and type(tc.UseTool) == "function" then
                    tc:UseTool(tool, "Activated")
                end
            end)
        end
        pcall(function()
            local Knit = require(RS.Knit.Knit)
            local svc = Knit.GetService("GraniteBlastService")
            if svc and svc.Activated then
                svc.Activated:Fire(tool, air)
            end
        end)
        pcall(function()
            RS.Knit.Knit.Services.GraniteBlastService.RE.Activated:FireServer(tool, air)
        end)
        -- Minimal hold — release ASAP so server sees short charge / less heat
        task.delay(0.04, function()
            if not tool or not tool.Parent then return end
            pcall(function()
                local Knit = require(RS.Knit.Knit)
                local tc = Knit.GetController("ToolController")
                if tc and type(tc.UseTool) == "function" then
                    tc:UseTool(tool, "Deactivated")
                end
            end)
            pcall(function()
                local Knit = require(RS.Knit.Knit)
                local svc = Knit.GetService("GraniteBlastService")
                if svc and svc.Deactivated then
                    -- false/air + tiny ratio hints (server may ignore extras)
                    svc.Deactivated:Fire(tool, air, 0.05)
                end
            end)
            pcall(function()
                RS.Knit.Knit.Services.GraniteBlastService.RE.Deactivated:FireServer(tool, air, 0.05)
            end)
            pcall(function()
                local Knit = require(RS.Knit.Knit)
                local tc = Knit.GetController("ToolController")
                if tc and type(tc.UseKey) == "function" then
                    tc:UseKey(1, "Deactivated")
                end
            end)
        end)
    end

    fireGraniteTap(opts.skipToolController)

    if opts.alreadyVented then
        task.delay(0.14, function()
            if not isRyuMoveset() then return end
            forceClearClientHeatFlags(lp.Character)
            cancelRyuRecovery(lp.Character)
            fireGraniteTap(true)
        end)
    end
    _G.FireGraniteBlast = fireGraniteBlastAllPaths
end

function fireRyuSilentVent()
    if not (_G.RyuState and _G.RyuState.SilentVent) then return end
    if not isRyuMoveset() then return end
    local now = tick()
    if now - lastRyuVentAt < 0.35 then return end  -- debounce, don't stack
    lastRyuVentAt = now
    -- Only send the remote. Animation cancel + no-stun start ONLY when the server
    -- confirms the special executed (Recovery animation detected by hookRyuRecoveryAnimator).
    pcall(function()
        RS.Knit.Knit.Services.RyuService.RE.RightActivated:FireServer()
    end)
    pcall(function()
        local Knit = require(RS.Knit.Knit)
        local tc = Knit.GetController("ToolController")
        if tc and type(tc.UseSetService) == "function" then
            local target = nil
            pcall(function() target = tc:GetTarget() end)
            tc:UseSetService("RightActivated", target)
        end
    end)
end

function hookRyuRecoveryAnimator(char)
    disconnectRyuRecoveryAnimConns()
    stopRyuSprintTrack()
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    local function bindAnimator(animator)
        if not animator then return end
        local c = animator.AnimationPlayed:Connect(function(track)
            if not (_G.RyuState and _G.RyuState.SilentVent) then return end
            local aid = track.Animation and track.Animation.AnimationId
            local name = string.lower(tostring(track.Name or ""))
            -- Only match exact Ryu R-move recovery names; broad "recover" match was
            -- triggering on skill animations (e.g. Second Helping) causing self-fling.
            local exactRecovery = (name == "recovery" or name == "recovery2" or name == "recovery3")
            if not (isRyuRecoveryAnimId(aid) or exactRecovery) then return end
            -- defer Stop/AdjustSpeed so we never nest AnimationPlayed
            task.defer(function()
                if not (_G.RyuState and _G.RyuState.SilentVent) then return end
                if not track or not track.IsPlaying then return end
                beginRyuRPhase()
                pcall(function()
                    track:AdjustSpeed(RYU_RECOVERY_SPEED)
                    local len = track.Length
                    if typeof(len) == "number" and len > 0.05 then
                        track.TimePosition = math.max(0, len - 0.03)
                    end
                    track:Stop(0)
                end)
                silenceRyuRecoverySounds(char)
                cancelRyuRecovery(char)
            end)
        end)
        table.insert(ryuRecoveryAnimConns, c)
    end
    local existing = hum:FindFirstChildOfClass("Animator")
    if existing then bindAnimator(existing) end
    local function stripStunIfRPhase(child)
        if not (isRyuRPhase() and _G.RyuState and _G.RyuState.SilentVent) then return end
        if child.Name == "Stun" or child.Name == "Knockback" or child.Name == "NoSprint" then
            task.defer(function()
                if isRyuRPhase() and child.Parent then
                    pcall(function() child:Destroy() end)
                end
            end)
        end
    end
    local cAdd = char.ChildAdded:Connect(stripStunIfRPhase)
    table.insert(ryuRecoveryAnimConns, cAdd)
    local info = char:FindFirstChild("Info")
    if info then
        local cInfo = info.ChildAdded:Connect(stripStunIfRPhase)
        table.insert(ryuRecoveryAnimConns, cInfo)
    end
    local cHum = hum.ChildAdded:Connect(function(child)
        if child:IsA("Animator") then bindAnimator(child) end
    end)
    table.insert(ryuRecoveryAnimConns, cHum)
    -- Kill Recovery SFX the moment they spawn during R phase
    local function onSoundAdded(desc)
        if not desc:IsA("Sound") then return end
        if not desc:IsDescendantOf(char) then return end
        if not (isRyuRPhase() and _G.RyuState and _G.RyuState.SilentVent) then return end
        if isRyuRecoverySoundId(desc.SoundId) or string.find(string.lower(desc.Name), "recover") then
            task.defer(function()
                pcall(function()
                    desc.Volume = 0
                    desc:Stop()
                    desc:Destroy()
                end)
            end)
        end
    end
    table.insert(ryuRecoveryAnimConns, char.DescendantAdded:Connect(onSoundAdded))
end

local function hookRyuRemoteFire()
    if _G._SosyRyuRemoteHooked then return end
    if typeof(hookmetamethod) ~= "function" or typeof(getnamecallmethod) ~= "function" then return end
    _G._SosyRyuRemoteHooked = true
    local cachedRE = nil
    local function getRyuRE()
        if cachedRE and cachedRE.Parent then return cachedRE end
        pcall(function()
            cachedRE = RS.Knit.Knit.Services.RyuService.RE.RightActivated
        end)
        return cachedRE
    end
    local function onRyuFire()
        -- Remote sent; do NOT start R phase or cancel here.
        -- hookRyuRecoveryAnimator triggers the cancel only when Recovery anim plays
        -- (= server actually accepted the special).
        if not (_G.RyuState and _G.RyuState.SilentVent and isRyuMoveset()) then return end
        lastRyuVentAt = tick()
    end
    local old
    local function handler(self, ...)
        local re = getRyuRE()
        if re and self == re and string.lower(getnamecallmethod()) == "fireserver" then
            task.defer(onRyuFire)
        end
        return old(self, ...)
    end
    if typeof(newcclosure) == "function" then handler = newcclosure(handler) end
    old = hookmetamethod(game, "__namecall", handler)
end

local ryuRKeyHooked = false
function hookRyuRKey()
    if ryuRKeyHooked then return end
    ryuRKeyHooked = true
    UIS.InputBegan:Connect(function(input, processed)
        if processed then return end
        if not isRyuMoveset() then return end
        if input.KeyCode == Enum.KeyCode.R then
            if not (_G.RyuState and _G.RyuState.SilentVent) then return end
            fireRyuSilentVent()
            return
        end
        -- Skill 1 hold must stay vanilla (ToolController Activated→hold→Deactivated).
        -- Do NOT auto-fire Granite here — that was forcing an instant tap.
    end)
end

function ensureRyuSilentVentLoop()
    if ryuSilentVentLoopConn then
        ryuSilentVentLoopConn:Disconnect()
        ryuSilentVentLoopConn = nil
    end
    disconnectRyuRecoveryAnimConns()
    stopRyuSprintTrack()
    if not (_G.RyuState and _G.RyuState.SilentVent) then return end
	-- The stock Ryu heat bar is created by RyuController.Overheat. Make sure our
	-- heat hook is installed before vent logic starts and do not hide/pin it.
	_G.RyuState.LocalHeatBar = true
	_G.RyuState.HideHeatBar = false
	pcall(hookRyuHeat)
	pcall(ensureRyuHeatLoop)
    hookRyuRemoteFire()
    hookRyuRecoveryAnimator(lp.Character)
    hookRyuRKey()
    local ventAcc = 0
    ryuSilentVentLoopConn = game:GetService("RunService").Heartbeat:Connect(function(dt)
        if not (_G.RyuState and _G.RyuState.SilentVent) then
            stopRyuSprintTrack()
            return
        end
        local char = lp.Character
        if not char or not isRyuMoveset() then
            stopRyuSprintTrack()
            return
        end
        if isRyuRPhase() then
            cancelRyuRecovery(char)
            updateRyuSprintDuringR(char)
        else
            stopRyuSprintTrack()
            -- Idle: don't burn CPU every frame
            ventAcc = ventAcc + dt
            if ventAcc < 0.2 then return end
            ventAcc = 0
        end
    end)
end

function forceZeroHeatObject(v)
    if not v or not v.Parent then return end
    if v:IsA("NumberValue") or v:IsA("IntValue") then
        if v.Value ~= 0 then
            v.Value = 0
        end
    end
end

-- Character.Info.Overheat is the heat meter (UI + server state). Client Overheat alone does NOT
-- block Granite on ToolController. Real R lock is Info.Stun after RightActivated / Recovery.
-- Pin only when HideHeatBar wants a fake empty bar.
function pinRyuOverheat()
    local char = lp.Character
    if not char then return end
    local info = char:FindFirstChild("Info")
    if not info then return end
    local oh = info:FindFirstChild("Overheat")
    if oh and (oh:IsA("NumberValue") or oh:IsA("IntValue")) and oh.Value ~= 0 then
        oh.Value = 0
    end
    -- Orange lock flag — client write helps UI; server still needs RightActivated
    local ohed = info:FindFirstChild("Overheated")
    if ohed and ohed:IsA("BoolValue") and ohed.Value then
        pcall(function() ohed.Value = false end)
    end
end

function hookOverheatInstance(oh)
    if not oh or not (oh:IsA("NumberValue") or oh:IsA("IntValue")) then return end
    -- Only zero the value when fully hiding (HideHeatBar AND NOT LocalHeatBar).
    -- LocalHeatBar shows the real fill so we must never clear it here.
    local wantPin = _G.RyuState and _G.RyuState.HideHeatBar and not (_G.RyuState.LocalHeatBar)
    if oh:GetAttribute("__SosyHeatHooked") then
        if wantPin and oh.Value ~= 0 then oh.Value = 0 end
        return
    end
    pcall(function() oh:SetAttribute("__SosyHeatHooked", true) end)
    if wantPin then oh.Value = 0 end
    local c = oh:GetPropertyChangedSignal("Value"):Connect(function()
        if not (_G.RyuState and _G.RyuState.HideHeatBar) then return end
        if _G.RyuState.LocalHeatBar then return end
        if oh.Value ~= 0 then
            oh.Value = 0
        end
    end)
    table.insert(ryuHeatValueHooks, c)
end

function zeroRyuHeatValues()
    local char = lp.Character
    if not char then return end
    -- Skip heavy work when not on Ryu
    if not isRyuMoveset() then return end
    local info = char:FindFirstChild("Info")
    local oh = info and info:FindFirstChild("Overheat")
    if oh then hookOverheatInstance(oh) end
    stripForeignRyuHeatUI()
    -- LocalHeatBar: keep own Buildup visible (don't wipe / don't pin every frame).
    if _G.RyuState and _G.RyuState.LocalHeatBar then
        return
    end
    pinRyuOverheat()
    cleanupRyuHeatUI(char)
end

function watchHeatAttributes()
    disconnectRyuAttrConns()
    local char = lp.Character
    if not char then return end
    local info = char:FindFirstChild("Info")
    if info then
        local oh = info:FindFirstChild("Overheat")
        if oh then hookOverheatInstance(oh) end
        local c = info.ChildAdded:Connect(function(child)
            if child.Name == "Overheat" then
                hookOverheatInstance(child)
            elseif child.Name == "Overheated" and child:IsA("BoolValue") then
                -- When bar goes orange, keep trying to clear client flag; skill still needs vent on 1
                local c2 = child:GetPropertyChangedSignal("Value"):Connect(function()
                    if child.Value and _G.RyuState and _G.RyuState.HideHeatBar then
                        pcall(function() child.Value = false end)
                    end
                end)
                table.insert(ryuHeatValueHooks, c2)
            end
        end)
        table.insert(ryuAttrConns, c)
    end
end

local ryuToolPatched = false
function patchToolControllerForOverheat()
    if ryuToolPatched then return end
    pcall(function()
        local Knit = require(RS.Knit.Knit)
        local tc = Knit.GetController("ToolController")
        if not tc or type(tc.UseTool) ~= "function" then return end
        ryuToolPatched = true
        local oldUse = tc.UseTool
        tc.UseTool = function(self, tool, action, ...)
            local toolName = typeof(tool) == "Instance" and tool.Name or tostring(tool)
            local isGranite = toolName == "Granite Blast" or string.find(string.lower(toolName), "granite")
            if isGranite and action == "Activated" and _G.RyuState and isRyuMoveset()
                and (_G.RyuState.SilentVent or _G.RyuState.HideHeatBar) then
                -- Orange lock: silent-vent first, then let hold continue if 1 is still down
                if isRyuOrange() and not ryuGraniteBusy then
                    task.spawn(function()
                        ryuGraniteBusy = true
                        ventClearOrangeForGranite()
                        local char = lp.Character
                        local deadline = tick() + 0.9
                        while tick() < deadline do
                            if not isRyuOrange() then break end
                            if char then
                                cancelRyuRecovery(char)
                                forceClearClientHeatFlags(char)
                            end
                            task.wait(0.03)
                        end
                        cancelRyuRecovery(lp.Character)
                        forceClearClientHeatFlags(lp.Character)
                        lastGraniteFireAt = 0
                        ryuGraniteBusy = false
                        -- Resume charge only while player still holding 1
                        local holding = UIS:IsKeyDown(Enum.KeyCode.One)
                            or UIS:IsKeyDown(Enum.KeyCode.KeypadOne)
                        if holding and tool and tool.Parent then
                            pcall(function()
                                oldUse(self, tool, "Activated")
                            end)
                        end
                    end)
                    return
                end
                if isRyuRPhase() then
                    cancelRyuRecovery(lp.Character)
                    forceClearClientHeatFlags(lp.Character)
                end
            end
            if _G.RyuState and _G.RyuState.HideHeatBar then
                pinRyuOverheat()
            end
            -- Pass through Activated/Deactivated normally so hold-to-charge works
            return oldUse(self, tool, action, ...)
        end
    end)
end

function ensureRyuHeatLoop()
    if ryuHeatLoopConn then
        ryuHeatLoopConn:Disconnect()
        ryuHeatLoopConn = nil
    end
    if _G.RyuState.HideHeatBar or _G.RyuState.LocalHeatBar then
        watchHeatAttributes()
        patchToolControllerForOverheat()
        local heatAcc = 0
        -- LocalHeatBar-only path mostly strips foreign UI — keep it slow
        local heatInterval = (_G.RyuState.HideHeatBar and not _G.RyuState.LocalHeatBar) and 0.15 or 0.45
        ryuHeatLoopConn = game:GetService("RunService").Heartbeat:Connect(function(dt)
            heatAcc = heatAcc + dt
            if heatAcc < heatInterval then return end
            heatAcc = 0
            zeroRyuHeatValues()
        end)
        -- warn("[SosyHUB][Ryu] Heat loop running")
    else
        disconnectRyuAttrConns()
    end
end

function hookRyuHeat()
    _G.RyuState = _G.RyuState or {}
    if _G.RyuState.Hooked then return end

    local KnitFolder = RS:FindFirstChild("Knit")
    if not KnitFolder then
        _G.RyuState.HookFailed = "Knit folder missing"
        return
    end
    local KnitModule = KnitFolder:FindFirstChild("Knit")
    if not KnitModule then
        _G.RyuState.HookFailed = "Knit module missing"
        return
    end

    local ok, Knit = pcall(require, KnitModule)
    if not (ok and Knit and type(Knit.GetController) == "function") then
        _G.RyuState.HookFailed = "Knit.GetController unavailable"
        return
    end

    local okCtrl, ryuCtrl = pcall(function() return Knit.GetController("RyuController") end)
    if not (okCtrl and ryuCtrl and ryuCtrl.Overheat) then
        _G.RyuState.HookFailed = "RyuController missing"
        warn("[SosyHUB][Ryu] Hook failed: RyuController missing")
        return
    end

    _G.RyuState.Hooked = true
    _G.RyuState.HookFailed = nil
    local originalOverheat = ryuCtrl.Overheat
    ryuCtrl.Overheat = function(self, heatValue)
        local isLocal = isLocalRyuCharacter(self)
        -- Never create / keep heat bar UI on other characters
        if _G.RyuState.LocalHeatBar and not isLocal then
            cleanupRyuHeatUI(self)
            return
        end
        if heatValue then
            hookOverheatInstance(heatValue)
        end
        -- Local-only bar: show real Overheat UI for me
        if _G.RyuState.LocalHeatBar and isLocal then
            return originalOverheat(self, heatValue)
        end
        if _G.RyuState.HideHeatBar then
            cleanupRyuHeatUI(self)
            pinRyuOverheat()
            return -- fully hidden + no R / Recovery
        end
        return originalOverheat(self, heatValue)
    end
    -- warn("[SosyHUB][Ryu] Hooked RyuController Overheat; LocalHeatBar=", _G.RyuState.LocalHeatBar)
end

-- ===================== Ryu respawn persistence =====================
-- Every Ryu hook except the RyuController/ToolController patches is bound to the
-- *current* character (Animator.AnimationPlayed, char.ChildAdded, Info.ChildAdded,
-- DescendantAdded). On death those instances go away with the old model and
-- nothing re-binds them, which is why the features had to be re-toggled by hand.
-- This watcher re-arms them on every new character while the toggles stay on.
function reapplyRyuForCharacter(char)
    char = char or lp.Character
    if not char then return end
    local st = _G.RyuState
    if not (st and (st.SilentVent or st.HideHeatBar or st.LocalHeatBar)) then return end

    -- per-life state must not carry over from the previous body
    ryuUnlockUntil = 0
    lastRyuVentAt = 0
    lastGraniteFireAt = 0
    ryuGraniteBusy = false
    stopRyuSprintTrack()

    if st.SilentVent then
        pcall(hookRyuRemoteFire)
        pcall(hookRyuRKey)
        pcall(hookRyuRecoveryAnimator, char)
    end
    if st.HideHeatBar or st.LocalHeatBar then
        pcall(hookRyuHeat)
        pcall(watchHeatAttributes)
        pcall(ensureRyuHeatLoop)
    end
end

function ensureRyuRespawnWatcher()
    if _G.__SosyRyuRespawnRunning then return end
    _G.__SosyRyuRespawnRunning = true

    if _G.__SosyRyuRespawnConn then
        pcall(function() _G.__SosyRyuRespawnConn:Disconnect() end)
    end
    _G.__SosyRyuRespawnConn = lp.CharacterAdded:Connect(function(char)
        task.spawn(function()
            local st = _G.RyuState
            if not (st and (st.SilentVent or st.HideHeatBar or st.LocalHeatBar)) then return end
            char:WaitForChild("Humanoid", 10)
            char:WaitForChild("HumanoidRootPart", 10)
            task.wait(0.35)
            if lp.Character ~= char then return end
            reapplyRyuForCharacter(char)
        end)
    end)

    -- Safety net: this game reparents characters into workspace.Characters and some
    -- deaths swap the model without CharacterAdded landing where we expect it, so
    -- also poll for "the character I hooked is not the character I have now".
    task.spawn(function()
        local hooked = lp.Character
        while _G.__SosyRyuRespawnRunning do
            task.wait(1)
            local st = _G.RyuState
            if st and (st.SilentVent or st.HideHeatBar or st.LocalHeatBar) then
                local cur = lp.Character
                if cur and cur ~= hooked and cur.Parent then
                    hooked = cur
                    pcall(reapplyRyuForCharacter, cur)
                elseif cur == nil then
                    hooked = nil
                end
            end
        end
    end)
end

-- Arm it as soon as the natives chunk loads; it no-ops until a Ryu toggle is on.
task.spawn(ensureRyuRespawnWatcher)

-- ===================== AUTO BLOCK (engine ported from xrezt) =====================
-- The previous native fired "any RemoteEvent whose name contains block" whenever a
-- player was within 15 studs. This game routes blocking through
-- ReplicatedStorage.Knit.Knit.Services.BlockService.RE.Activated / .Deactivated, so
-- that never did anything. This is the xrezt auto block module (state machine,
-- attack-animation registry, projectile scanner, hit/damage reaction) rewired to
-- read _G.AutoBlockState instead of the Linoria Toggles/Options tables.
_G.SosyAutoBlock = (function()
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = workspace
local LocalPlayer = Players.LocalPlayer

local AB = {}
local Running = false
local Conns = {}

local function addConn(c)
    if c then Conns[#Conns + 1] = c end
    return c
end

local function dropConns()
    for i = #Conns, 1, -1 do
        pcall(function() Conns[i]:Disconnect() end)
        Conns[i] = nil
    end
end

-- ── config bridge ────────────────────────────────────────────────────────────
-- Mutated in place (never re-allocated) because it is read from Heartbeat.
local C = {
    enabled = false, trail = false, move = false, proj = false,
    whileAttacking = false, onlyLocked = false,
    trailDist = 13, trailDot = 0.26, moveRange = 12, moveFacing = 0.7,
    projRange = 25, projFacing = 0.4,
    duration = 0.5, minHold = 0.06, cooldown = 0, releaseBuffer = 0.05,
    damageThreshold = 5, damageHold = 0.4,
    antiFeint = true, hitGlow = true, damageReaction = true,
}
local lastCfgAt = 0

local function nz(v, d)
    v = tonumber(v)
    if v == nil then return d end
    return v
end

local function refreshCfg(force)
    local now = tick()
    if not force and now - lastCfgAt < 0.1 then return C end
    lastCfgAt = now

    local s = _G.AutoBlockState
    if type(s) ~= "table" then s = {} ; _G.AutoBlockState = s end

    local trail, move, proj = s.Trail == true, s.MoveAnim == true, s.Projectile == true
    -- Mobile "Block" button / master toggle only flips Enabled: treat that as all-on.
    if s.Enabled and not (trail or move or proj) then
        trail, move, proj = true, true, true
    end

    C.trail, C.move, C.proj = trail, move, proj
    C.enabled = (s.Enabled == true) and (trail or move or proj)
    C.whileAttacking = s.WhileAttacking == true
    C.onlyLocked = s.OnlyLocked == true

    C.trailDist = nz(s.TrailDistance, 13)
    local ang = math.clamp(nz(s.TrailAngle, 75), 0, 180)
    C.trailDot = math.cos(math.rad(ang))
    C.moveRange = nz(s.MoveRange, 12)
    C.moveFacing = nz(s.MoveFacing, 0.7)
    C.projRange = nz(s.ProjectileRange, 25)
    C.projFacing = nz(s.ProjectileFacing, 0.4)

    C.duration = nz(s.BlockDuration, 0.5)
    C.minHold = nz(s.MinHold, 0.06)
    C.cooldown = nz(s.Cooldown, 0)
    C.releaseBuffer = nz(s.ReleaseBuffer, 0.05)
    C.damageThreshold = nz(s.DamageThreshold, 5)
    C.damageHold = nz(s.DamageHold, 0.4)

    C.antiFeint = s.AntiFeint ~= false
    C.hitGlow = s.HitGlow ~= false
    C.damageReaction = s.DamageReaction ~= false
    return C
end

-- ── remotes ──────────────────────────────────────────────────────────────────
local BlockRemoteCache = {
    _activated = nil,
    _deactivated = nil,
    _resolved = false,
    _lastTry = 0,
}

function BlockRemoteCache:Resolve()
    if self._resolved then return true end
    local now = tick()
    if now - self._lastTry < 2 then return false end
    self._lastTry = now
    pcall(function()
        local knit = ReplicatedStorage:WaitForChild("Knit", 3)
        local inner = knit:WaitForChild("Knit", 3)
        local services = inner:WaitForChild("Services", 3)
        local blockService = services:FindFirstChild("BlockService")
        if blockService then
            local re = blockService:FindFirstChild("RE")
            if re then
                self._activated = re:FindFirstChild("Activated")
                self._deactivated = re:FindFirstChild("Deactivated")
                if self._activated and self._deactivated then
                    self._resolved = true
                end
            end
        end
    end)
    return self._resolved
end

function BlockRemoteCache:GetActivated()
    if not self._resolved then self:Resolve() end
    return self._activated
end

function BlockRemoteCache:GetDeactivated()
    if not self._resolved then self:Resolve() end
    return self._deactivated
end

-- ── block state machine ──────────────────────────────────────────────────────
local BlockStateMachine = {
    _activeReasons = {},
    _nextId = 0,
    _isBlocking = false,
    _lastBlockTime = 0,
    _releaseTime = 0,
    _totalActivated = 0,
    _totalReleased = 0,
    _lastReason = "",
    _blockStartTime = 0,
    _minHoldThread = nil,
}

function BlockStateMachine:_activateBlock()
    if self._isBlocking then return end
    local remote = BlockRemoteCache:GetActivated()
    if remote then
        pcall(function() remote:FireServer() end)
    end
    self._isBlocking = true
    self._blockStartTime = tick()
    self._totalActivated = self._totalActivated + 1
end

function BlockStateMachine:_deactivateBlock()
    if not self._isBlocking then return end
    local remote = BlockRemoteCache:GetDeactivated()
    if remote then
        pcall(function() remote:FireServer() end)
    end
    self._isBlocking = false
    self._releaseTime = tick()
    self._totalReleased = self._totalReleased + 1
end

function BlockStateMachine:Request(reason, duration)
    self._nextId = self._nextId + 1
    local id = self._nextId
    local now = tick()

    self._activeReasons[id] = {
        reason = reason or "unknown",
        createdAt = now,
        expiresAt = duration and (now + duration) or nil,
    }

    if not self._isBlocking then
        local cooldown = C.cooldown
        local sinceRelease = now - self._releaseTime
        if sinceRelease < cooldown then
            task.delay(cooldown - sinceRelease, function()
                if not self._isBlocking then
                    local stillHasReasons = false
                    for _ in pairs(self._activeReasons) do stillHasReasons = true break end
                    if stillHasReasons then self:_activateBlock() end
                end
            end)
        else
            self:_activateBlock()
        end
    end

    self._lastBlockTime = now
    self._lastReason = reason or "unknown"
    return id
end

function BlockStateMachine:_evaluateState()
    local now = tick()
    local hasAlive = false

    for id, data in pairs(self._activeReasons) do
        if data.expiresAt and now > data.expiresAt then
            self._activeReasons[id] = nil
        else
            hasAlive = true
        end
    end

    if not hasAlive and self._isBlocking then
        local holdTime = now - self._lastBlockTime
        local minHold = C.minHold
        if holdTime < minHold then
            if self._minHoldThread then
                pcall(task.cancel, self._minHoldThread)
            end
            self._minHoldThread = task.delay(minHold - holdTime, function()
                self._minHoldThread = nil
                local stillEmpty = true
                for _ in pairs(self._activeReasons) do stillEmpty = false break end
                if stillEmpty and self._isBlocking then
                    self:_deactivateBlock()
                end
            end)
        else
            self:_deactivateBlock()
        end
    end
end

function BlockStateMachine:Release(id)
    if id and self._activeReasons[id] then
        self._activeReasons[id] = nil
    end
    self:_evaluateState()
end

function BlockStateMachine:ReleaseAll()
    table.clear(self._activeReasons)
    if self._isBlocking then
        self:_deactivateBlock()
    end
end

function BlockStateMachine:IsBlocking()
    return self._isBlocking
end

function BlockStateMachine:GetActiveCount()
    local count = 0
    for _ in pairs(self._activeReasons) do count = count + 1 end
    return count
end

-- ── attack animation registry ────────────────────────────────────────────────
local AttackAnimationRegistry = {}
local R = AttackAnimationRegistry

R["132748613906344"] = { name = "Gojo.HollowPurple", range = 80, duration = 1.8, isProjectile = true, priority = 10 }
R["137654778575373"] = { name = "Gojo.ReversalRed", range = 60, duration = 0.8, isProjectile = true, priority = 8 }
R["137865634124104"] = { name = "Gojo.LapseBlue", range = 50, duration = 0.7, isProjectile = true, priority = 8 }
R["101162958113766"] = { name = "Gojo.LapseBlueMaximum", range = 60, duration = 1.0, isProjectile = true, priority = 9 }
R["84716311536982"]  = { name = "Gojo.UltimatePurple", range = 100, duration = 2.0, isProjectile = true, priority = 10 }
R["132725601768618"] = { name = "Gojo.InfiniteVoid", range = 100, duration = 2.0, isProjectile = true, priority = 10 }
R["95421145178968"]  = { name = "Gojo.RapidPunches", range = 20, duration = 1.5, isProjectile = false, priority = 7 }
R["127851700400958"] = { name = "Gojo.Melee1", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["72548435296350"]  = { name = "Gojo.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["84547415708554"]  = { name = "Gojo.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["124937162378188"] = { name = "Gojo.ReversalRedMaximum", range = 65, duration = 1.2, isProjectile = true, priority = 9 }
R["104749346956269"] = { name = "Gojo.TwofoldKick", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["132673356426065"] = { name = "Gojo.TwofoldHit", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["116811846715462"] = { name = "Gojo.ShortVoid", range = 50, duration = 1.0, isProjectile = false, priority = 8 }

R["77200218033775"]  = { name = "Itadori.CursedStrike", range = 18, duration = 0.5, isProjectile = false, priority = 5 }
R["124901309160375"] = { name = "Itadori.CrushingBlow", range = 18, duration = 0.5, isProjectile = false, priority = 5 }
R["82987093810211"]  = { name = "Itadori.ManjiKick", range = 16, duration = 0.5, isProjectile = false, priority = 5 }
R["121923107958102"] = { name = "Itadori.SlaughterDemon", range = 18, duration = 0.5, isProjectile = false, priority = 6 }
R["107554693613496"] = { name = "Itadori.Rush", range = 22, duration = 0.8, isProjectile = false, priority = 6 }
R["110146909061402"] = { name = "Itadori.Melee1", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["123414935051274"] = { name = "Itadori.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["108636011034323"] = { name = "Itadori.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["105376952884290"] = { name = "Itadori.Melee4", range = 15, duration = 0.40, isProjectile = false, isM1 = true, priority = 3 }
R["100962226150441"] = { name = "Itadori.DivergentFist1", range = 16, duration = 0.45, isProjectile = false, priority = 5 }
R["95852624447551"]  = { name = "Itadori.DivergentFist2", range = 16, duration = 0.45, isProjectile = false, priority = 5 }
R["74145636023952"]  = { name = "Itadori.DivergentFist3", range = 16, duration = 0.45, isProjectile = false, priority = 5 }
R["123171106092050"] = { name = "Itadori.DivergentFist4", range = 16, duration = 0.45, isProjectile = false, priority = 5 }
R["81633998750531"]  = { name = "Itadori.UltimateEnchain", range = 18, duration = 0.5, isProjectile = false, priority = 7 }
R["130206074036010"] = { name = "Itadori.Instincts", range = 20, duration = 0.5, isProjectile = false, priority = 6 }

R["111593784328268"] = { name = "Sukuna.Cleave", range = 20, duration = 0.6, isProjectile = false, priority = 7 }
R["131506102901134"] = { name = "Sukuna.Dismantle", range = 45, duration = 0.6, isProjectile = true, priority = 8 }
R["137611726964398"] = { name = "Sukuna.FlameArrow", range = 80, duration = 1.5, isProjectile = true, priority = 9 }
R["121984128639453"] = { name = "Sukuna.MalevolentShrine", range = 100, duration = 2.0, isProjectile = false, priority = 10 }
R["95295463826732"]  = { name = "Heian.Melee1", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["105077924973072"] = { name = "Heian.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["124862357369335"] = { name = "Heian.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["81630213087988"]  = { name = "Heian.Melee4", range = 15, duration = 0.40, isProjectile = false, isM1 = true, priority = 3 }

R["73243807139765"]  = { name = "Ryu.GraniteBlast", range = 120, duration = 1.5, isProjectile = true, priority = 10 }
R["70394890117813"]  = { name = "Ryu.Appetizer", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["138826705245289"] = { name = "Ryu.SecondHelping", range = 25, duration = 0.7, isProjectile = false, priority = 6 }
R["131917532383382"] = { name = "Ryu.ThisDessert", range = 25, duration = 0.7, isProjectile = false, priority = 6 }
R["86568794583359"]  = { name = "Ryu.WhatAreYouAfter", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["86463226245064"]  = { name = "Ryu.WerentInvited", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["116910683335467"] = { name = "Ryu.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["121322029260156"] = { name = "Ryu.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["92698956945928"]  = { name = "Ryu.Melee3_Alt", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["113479860283691"] = { name = "Ryu.UltimateBlast", range = 100, duration = 2.0, isProjectile = true, priority = 10 }

R["127171275866632"] = { name = "Choso.PiercingBlood", range = 100, duration = 1.0, isProjectile = true, priority = 9 }
R["85569553424083"]  = { name = "Choso.Supernova", range = 30, duration = 0.8, isProjectile = true, priority = 7 }
R["100446064103831"] = { name = "Choso.BloodEdge", range = 20, duration = 0.5, isProjectile = false, priority = 5 }
R["132928484483887"] = { name = "Choso.Hairpin", range = 40, duration = 0.6, isProjectile = true, priority = 7 }
R["114321791577837"] = { name = "Choso.SlicingExorcism", range = 25, duration = 0.6, isProjectile = false, priority = 6 }
R["117371289990421"] = { name = "Choso.BloodRain", range = 35, duration = 0.8, isProjectile = false, priority = 7 }
R["95097480425566"]  = { name = "Choso.WingKing", range = 50, duration = 1.2, isProjectile = true, priority = 8 }
R["119042572747325"] = { name = "Choso.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["105287938257399"] = { name = "Choso.Melee4", range = 15, duration = 0.40, isProjectile = false, isM1 = true, priority = 3 }

R["72063002791216"]  = { name = "Hakari.ShutterDoors", range = 25, duration = 0.6, isProjectile = true, priority = 6 }
R["72467492674240"]  = { name = "Hakari.RoughEnergy", range = 20, duration = 0.5, isProjectile = false, priority = 5 }
R["82541714192027"]  = { name = "Hakari.ReserveBalls", range = 40, duration = 0.6, isProjectile = true, priority = 7 }
R["95901746347992"]  = { name = "Hakari.LuckyVolley", range = 30, duration = 0.6, isProjectile = false, priority = 6 }
R["108123475959041"] = { name = "Hakari.FeverBreaker", range = 25, duration = 0.7, isProjectile = false, priority = 6 }
R["140588454098230"] = { name = "Hakari.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["138826758216894"] = { name = "Hakari.Melee4", range = 15, duration = 0.40, isProjectile = false, isM1 = true, priority = 3 }

R["116432619539029"] = { name = "Megumi.Toad", range = 40, duration = 0.6, isProjectile = true, priority = 7 }
R["111077341852080"] = { name = "Megumi.Nue", range = 40, duration = 0.7, isProjectile = true, priority = 7 }
R["81112033595734"]  = { name = "Megumi.DivineDog", range = 30, duration = 0.6, isProjectile = true, priority = 6 }
R["75390215999547"]  = { name = "Megumi.GreatSerpent", range = 35, duration = 0.6, isProjectile = false, priority = 6 }
R["115683433001643"] = { name = "Megumi.DivinePummel", range = 25, duration = 0.8, isProjectile = false, priority = 7 }
R["138852224035589"] = { name = "Megumi.GroundPitch", range = 30, duration = 0.6, isProjectile = false, priority = 6 }
R["85024950165903"]  = { name = "Megumi.Earthquake", range = 30, duration = 0.7, isProjectile = false, priority = 7 }
R["131219281339199"] = { name = "Megumi.MaxElephant", range = 40, duration = 0.8, isProjectile = true, priority = 8 }
R["138489871864252"] = { name = "Megumi.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

R["89092734635186"]  = { name = "Mahito.Soulfire", range = 30, duration = 0.6, isProjectile = true, priority = 7 }
R["127727754867974"] = { name = "Mahito.SpikeWrath", range = 25, duration = 0.7, isProjectile = false, priority = 6 }
R["105068005007692"] = { name = "Mahito.FaceBlitz", range = 20, duration = 0.5, isProjectile = false, priority = 5 }
R["108319980293313"] = { name = "Mahito.BodyRepel", range = 30, duration = 0.7, isProjectile = false, priority = 6 }
R["76313364850487"]  = { name = "Mahito.WideStrike", range = 25, duration = 0.6, isProjectile = false, priority = 6 }
R["94223344057046"]  = { name = "Mahito.HeartPiercer", range = 22, duration = 0.6, isProjectile = false, priority = 6 }
R["128779949980528"] = { name = "Mahito.DrillSplit", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["86073608599582"]  = { name = "Mahito.HeadSplitter", range = 20, duration = 0.5, isProjectile = false, priority = 5 }

R["94720627091769"]  = { name = "Todo.SwiftKick", range = 18, duration = 0.5, isProjectile = false, priority = 5 }
R["111720035828971"] = { name = "Todo.PebbleThrow", range = 50, duration = 0.6, isProjectile = true, priority = 7 }
R["136536827155962"] = { name = "Todo.BruteForce", range = 15, duration = 0.5, isProjectile = false, priority = 5 }
R["121343824534765"] = { name = "Todo.ElbowDrop", range = 18, duration = 0.6, isProjectile = false, priority = 5 }
R["107029561762376"] = { name = "Todo.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["117831239064143"] = { name = "Todo.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

R["124340599144108"] = { name = "Hiromi.TwirlingStrikes", range = 18, duration = 0.6, isProjectile = false, priority = 5 }
R["132754851925571"] = { name = "Hiromi.Verdict", range = 20, duration = 0.5, isProjectile = false, priority = 5 }
R["86362077638309"]  = { name = "Hiromi.GavelThrow", range = 40, duration = 0.6, isProjectile = true, priority = 7 }
R["124243904748268"] = { name = "Hiromi.TripleSentence", range = 22, duration = 0.7, isProjectile = false, priority = 6 }
R["124759375124281"] = { name = "Hiromi.JudgeReach", range = 25, duration = 0.6, isProjectile = false, priority = 6 }
R["122573730331631"] = { name = "Hiromi.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["82400997593751"]  = { name = "Hiromi.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["118634493886688"] = { name = "Hiromi.Melee4", range = 15, duration = 0.40, isProjectile = false, isM1 = true, priority = 3 }

R["80465501985014"]  = { name = "Mechamaru.MiracleCannon", range = 80, duration = 1.0, isProjectile = true, priority = 9 }
R["118652212972529"] = { name = "Mechamaru.GunShot", range = 60, duration = 0.6, isProjectile = true, priority = 8 }
R["93901924492394"]  = { name = "Mechamaru.UltraCannon", range = 100, duration = 1.5, isProjectile = true, priority = 10 }
R["137638103122538"] = { name = "Mechamaru.AbsoluteDestruction", range = 100, duration = 2.0, isProjectile = true, priority = 10 }
R["89009042593684"]  = { name = "Mechamaru.PigeonViola", range = 70, duration = 1.0, isProjectile = true, priority = 8 }
R["114277419400774"] = { name = "Mechamaru.HeatEmission", range = 30, duration = 0.7, isProjectile = false, priority = 6 }
R["85148168523745"]  = { name = "Mechamaru.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["108686045412945"] = { name = "Mechamaru.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

R["104793932628579"] = { name = "Yuki.MassBreaker", range = 25, duration = 0.7, isProjectile = false, priority = 6 }
R["94347210073500"]  = { name = "Yuki.RisingStar", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["77833820443705"]  = { name = "Yuki.GarudaStab", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["115097960689033"] = { name = "Yuki.GarudaRebound", range = 25, duration = 0.6, isProjectile = false, priority = 6 }
R["72575786212990"]  = { name = "Yuki.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["119248903710146"] = { name = "Yuki.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

R["81971779090581"]  = { name = "Goku.Kamehameha", range = 120, duration = 1.5, isProjectile = true, priority = 10 }
R["87481059409847"]  = { name = "Goku.StaffExtend", range = 50, duration = 0.6, isProjectile = true, priority = 7 }
R["128537969081721"] = { name = "Goku.KiSpam", range = 80, duration = 1.5, isProjectile = true, priority = 9 }
R["117318845383884"] = { name = "Goku.StaffUppercut", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["97215638330770"]  = { name = "Goku.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["100474683542881"] = { name = "Goku.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

R["89582140026963"]  = { name = "Yuta.ResoluteSlash", range = 25, duration = 0.6, isProjectile = false, priority = 6 }
R["116737527523542"] = { name = "Yuta.LoveBeam", range = 80, duration = 1.2, isProjectile = true, priority = 9 }
R["74676954665401"]  = { name = "Yuta.Slam", range = 25, duration = 0.6, isProjectile = false, priority = 6 }
R["95169958463123"]  = { name = "Yuta.Throw", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["140288981168553"] = { name = "Yuta.Smash", range = 25, duration = 0.6, isProjectile = false, priority = 6 }
R["73482562876920"]  = { name = "Yuta.JacobLadder", range = 60, duration = 1.5, isProjectile = true, priority = 9 }
R["88005970155216"]  = { name = "Yuta.CursedSpeech", range = 35, duration = 0.8, isProjectile = false, priority = 7 }
R["130806585141471"] = { name = "Yuta.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["131967150738931"] = { name = "Yuta.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

R["105121164520635"] = { name = "Naoya.ProjectionBreaker", range = 25, duration = 0.6, isProjectile = false, priority = 6 }
R["122607727974119"] = { name = "Naoya.Flicker", range = 20, duration = 0.5, isProjectile = false, priority = 5 }
R["86045680364061"]  = { name = "Naoya.DecisiveStrike", range = 20, duration = 0.5, isProjectile = false, priority = 5 }
R["129944486689528"] = { name = "Naoya.Acceleration", range = 30, duration = 0.8, isProjectile = false, priority = 7 }
R["108708446862011"] = { name = "Naoya.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["77583711129628"]  = { name = "Naoya.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

R["100811576955331"] = { name = "Nanami.BluntCut", range = 18, duration = 0.5, isProjectile = false, priority = 5 }
R["122015481201264"] = { name = "Nanami.Collapse", range = 25, duration = 0.7, isProjectile = false, priority = 6 }
R["130957217409359"] = { name = "Nanami.SeveranceKick", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["79436586236026"]  = { name = "Nanami.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }
R["102285403332509"] = { name = "Nanami.Melee3", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

R["103960582499076"] = { name = "Haruta.AnkleCutter", range = 15, duration = 0.5, isProjectile = false, priority = 5 }
R["118326207788271"] = { name = "Haruta.Jawbreaker", range = 15, duration = 0.5, isProjectile = false, priority = 5 }
R["133303451091615"] = { name = "Haruta.Backstab", range = 16, duration = 0.5, isProjectile = false, priority = 5 }
R["113963875117859"] = { name = "Haruta.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

R["139956651661073"] = { name = "MeiMei.Bounding", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["99180695169591"]  = { name = "MeiMei.BirdCall", range = 50, duration = 0.7, isProjectile = true, priority = 7 }
R["108449614447004"] = { name = "MeiMei.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

R["113722638806911"] = { name = "Hanami.RootSwarm", range = 35, duration = 0.7, isProjectile = false, priority = 6 }
R["96466374346823"]  = { name = "Hanami.BudShot", range = 50, duration = 0.6, isProjectile = true, priority = 7 }
R["92595499555055"]  = { name = "Hanami.SurgingThorns", range = 30, duration = 0.7, isProjectile = false, priority = 6 }
R["88849926869776"]  = { name = "Hanami.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

R["116119661056362"] = { name = "Kurourushi.RoachSwarm", range = 40, duration = 0.8, isProjectile = true, priority = 7 }
R["83430571986421"]  = { name = "Kurourushi.FesteringStrikes", range = 20, duration = 0.6, isProjectile = false, priority = 5 }
R["93751660565233"]  = { name = "Kurourushi.EarthenTrance", range = 30, duration = 0.7, isProjectile = false, priority = 6 }
R["86519781516542"]  = { name = "Kurourushi.Melee2", range = 15, duration = 0.35, isProjectile = false, isM1 = true, priority = 3 }

local LongRangeAttackIds = {
    ["132748613906344"] = true, ["84716311536982"] = true, ["132725601768618"] = true,
    ["137654778575373"] = true, ["137865634124104"] = true, ["101162958113766"] = true,
    ["124937162378188"] = true, ["73243807139765"] = true, ["113479860283691"] = true,
    ["127171275866632"] = true, ["95097480425566"] = true, ["131506102901134"] = true,
    ["137611726964398"] = true, ["121984128639453"] = true, ["80465501985014"] = true,
    ["93901924492394"] = true, ["137638103122538"] = true, ["89009042593684"] = true,
    ["118652212972529"] = true, ["81971779090581"] = true, ["128537969081721"] = true,
    ["116737527523542"] = true, ["73482562876920"] = true, ["111720035828971"] = true,
    ["86362077638309"] = true, ["99180695169591"] = true, ["96466374346823"] = true,
    ["116119661056362"] = true, ["87481059409847"] = true, ["132928484483887"] = true,
    ["82541714192027"] = true, ["116432619539029"] = true, ["111077341852080"] = true,
    ["81112033595734"] = true, ["131219281339199"] = true, ["89092734635186"] = true,
    ["85569553424083"] = true, ["72063002791216"] = true,
}

local LongRangeTimingOverrides = {
    ["132748613906344"] = 2.00, ["84716311536982"]  = 2.20, ["132725601768618"] = 2.20,
    ["73243807139765"]  = 1.70, ["113479860283691"] = 2.20, ["127171275866632"] = 1.20,
    ["137611726964398"] = 1.70, ["121984128639453"] = 2.50, ["80465501985014"]  = 1.20,
    ["93901924492394"]  = 1.70, ["137638103122538"] = 2.20, ["81971779090581"]  = 1.70,
    ["128537969081721"] = 1.70, ["116737527523542"] = 1.40, ["73482562876920"]  = 1.70,
    ["131506102901134"] = 0.80, ["89009042593684"]  = 1.20, ["118652212972529"] = 0.80,
}

local MovementFilterKeywords = {
    "run", "chase", "down", "fall", "ragdoll", "idle",
    "walk", "jump", "dash", "climb", "getup", "land",
    "sprint", "movement", "turn", "halt", "hover", "sleep",
    "emote", "spawn", "dance", "wave", "sit", "block",
    "equip", "unequip", "sheath", "unsheath", "holster",
    "swim", "freefall", "climbidle", "crouch", "prone",
    "stun", "recover", "wind", "breathe", "loop", "breathing",
    "aura", "charge", "power", "transform", "shift", "activate", "idleloop",
    "stance", "guard", "parry", "counter", "deflect", "evade",
    "awaken", "mode", "form", "intro", "outro", "cinematic",
    "taunt", "celebrate", "victory", "defeat", "death", "die",
    "pickup", "interact", "use", "consume", "eat", "drink",
    "revive", "respawn", "tp", "teleport", "warp", "phase",
    "float", "fly", "glide", "soar", "levitate", "ascend",
    "descend", "mount", "dismount", "ride", "vehicle", "board",
}

local NonAttackPatterns = {
    "effect", "particle", "sound", "camera", "shake",
    "flash", "screen", "ui", "menu", "cursor",
    "blink", "talk", "look", "point", "nod", "shakehead",
    "shrug", "grab", "carry", "hold", "lift", "push", "pull",
    "heal", "revive", "buff", "debuff", "shield",
    "barrier", "teleport", "dodge", "roll", "slide",
    "grapple", "web", "swing", "success", "user",
    "victim", "hit", "warn", "stun", "ragdoll", "knockback", "reaction", "stagger",
    "notification", "alert", "popup", "toast", "badge",
    "loading", "progress", "bar", "meter", "gauge",
    "footstep", "cloth", "hair", "cape", "tail", "wing",
    "ambient", "environment", "weather", "rain", "snow",
    "expression", "face", "eye", "mouth", "brow",
    "gesture", "signal", "callout", "ping", "mark",
    "overlay", "hud", "indicator", "reticle", "crosshair",
}

local AttackIndicatorKeywords = {
    "attack", "slash", "strike", "kick", "punch", "melee",
    "swing", "smash", "slam", "crush", "cleave", "cut",
    "stab", "pierce", "thrust", "jab", "hook", "uppercut",
    "combo", "finisher", "barrage", "flurry", "rush",
    "shoot", "fire", "blast", "beam", "cannon", "shot",
    "throw", "toss", "hurl", "launch", "fling",
    "skill", "ability", "special", "ultimate", "ult",
    "m1", "m2", "heavy", "light",
    "damage", "hurt", "pain", "destroy", "break",
    "explode", "detonate", "burst", "shatter", "crack",
    "bite", "claw", "rip", "tear", "rend",
    "whip", "lash", "flail", "batter", "pummel",
    "chop", "hack", "slice", "dice", "mince",
    "impale", "skewer", "gore", "maul", "mangle",
    "bombard", "volley", "salvo", "fusillade", "rain",
    "decimate", "annihilate", "obliterate", "eradicate",
}

local DynamicAnimationCache = {}
local DynamicCacheMaxSize = 2048

local function PurgeOldestCacheEntries()
    local count = 0
    for _ in pairs(DynamicAnimationCache) do count = count + 1 end
    if count < DynamicCacheMaxSize then return end
    local entries = {}
    for id, data in pairs(DynamicAnimationCache) do
        entries[#entries + 1] = { id = id, lastSeen = data.lastSeen or 0 }
    end
    table.sort(entries, function(a, b) return a.lastSeen < b.lastSeen end)
    for i = 1, math.floor(#entries * 0.3) do
        DynamicAnimationCache[entries[i].id] = nil
    end
end

-- ── character helpers ────────────────────────────────────────────────────────
local function GetCharactersFolder()
    return Workspace:FindFirstChild("Characters")
end

local function GetLocalCharacter()
    local folder = GetCharactersFolder()
    local c = folder and folder:FindFirstChild(LocalPlayer.Name)
    if c and c:IsA("Model") then return c end
    return LocalPlayer.Character
end

local function GetLocalRoot()
    local char = GetLocalCharacter()
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function GetLocalHumanoid()
    local char = GetLocalCharacter()
    return char and char:FindFirstChildOfClass("Humanoid")
end

local function IsLocalPlayerIncapacitated()
    local char = GetLocalCharacter()
    if not char then return true end
    if char:GetAttribute("Dead") then return true end
    local ragdollVal = char:GetAttribute("Ragdoll")
    if ragdollVal and tonumber(ragdollVal) and tonumber(ragdollVal) > 0 then return true end
    if char:GetAttribute("Stun") == true then return true end
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if humanoid and humanoid.Health <= 0 then return true end
    return false
end

local function IsCharacterAlive(character)
    if not character or not character.Parent then return false end
    if character:GetAttribute("Dead") then return false end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if humanoid and humanoid.Health <= 0 then return false end
    return true
end

local function IsCharacterIncapacitated(character)
    if not character then return true end
    if character:GetAttribute("Dead") then return true end
    local ragdollVal = character:GetAttribute("Ragdoll")
    if ragdollVal and tonumber(ragdollVal) and tonumber(ragdollVal) > 0 then return true end
    if character:GetAttribute("Stun") == true then return true end
    return false
end

local function IsLocalPlayerAttacking()
    local char = GetLocalCharacter()
    if not char then return false end
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid then return false end
    local animator = humanoid:FindFirstChildOfClass("Animator")
    if not animator then return false end

    for _, track in pairs(animator:GetPlayingAnimationTracks()) do
        if track.IsPlaying and track.Animation then
            local animName = string.lower(track.Animation.Name)
            local rawId = tostring(track.Animation.AnimationId):match("%d+")
            if rawId and AttackAnimationRegistry[rawId] then
                return true
            end
            local isMovement = false
            for i = 1, #MovementFilterKeywords do
                if string.find(animName, MovementFilterKeywords[i], 1, true) then
                    isMovement = true
                    break
                end
            end
            if not isMovement then
                local isNonAttack = false
                for i = 1, #NonAttackPatterns do
                    if string.find(animName, NonAttackPatterns[i], 1, true) then
                        isNonAttack = true
                        break
                    end
                end
                if not isNonAttack then
                    for i = 1, #AttackIndicatorKeywords do
                        if string.find(animName, AttackIndicatorKeywords[i], 1, true) then
                            return true
                        end
                    end
                end
            end
        end
    end
    return false
end

local function GetDistanceBetween(rootA, rootB)
    if not rootA or not rootB then return math.huge end
    return (rootA.Position - rootB.Position).Magnitude
end

-- Dot of my look vector against the direction to the attacker: blocking only
-- mitigates damage from the front, so we require that I am facing them.
local function GetFacingDot(myRoot, theirRoot)
    if not myRoot or not theirRoot then return -1 end
    local delta = theirRoot.Position - myRoot.Position
    if delta.Magnitude < 0.001 then return 1 end
    return myRoot.CFrame.LookVector:Dot(delta.Unit)
end

local function GetVelocityTowardTarget(partPos, partVelocity, targetPos)
    if not partPos or not partVelocity or not targetPos then return 0, -1 end
    local speed = partVelocity.Magnitude
    if speed < 1 then return 0, -1 end
    return speed, partVelocity.Unit:Dot((targetPos - partPos).Unit)
end

local function IsEnemyFeinting(character)
    if not C.antiFeint then return false end
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return false end

    local feintInstance = rootPart:FindFirstChild("Feint")
    if feintInstance then
        local ringParticle = feintInstance:FindFirstChild("Ring")
        if ringParticle and ringParticle:IsA("ParticleEmitter") then
            for _, keypoint in ipairs(ringParticle.Color.Keypoints) do
                local val = keypoint.Value
                if math.abs(val.R - val.G) < 0.05 and math.abs(val.G - val.B) < 0.05 then
                    return true
                end
            end
        end
    end

    if character:GetAttribute("Feinting") then return true end

    local moveset = character:FindFirstChild("Moveset")
    if moveset then
        local feintVal = moveset:FindFirstChild("Feint")
        if feintVal and feintVal:IsA("BoolValue") and feintVal.Value then
            return true
        end
    end
    return false
end

local function ClassifyAnimation(animationTrack)
    if not animationTrack or not animationTrack.Animation then
        return false, false, nil
    end

    local animId = animationTrack.Animation.AnimationId
    local animName = animationTrack.Animation.Name
    local lowerName = string.lower(animName)
    local rawId = tostring(animId):match("%d+")

    if rawId and AttackAnimationRegistry[rawId] then
        local reg = AttackAnimationRegistry[rawId]
        return true, reg.isProjectile or false, reg
    end

    if rawId and DynamicAnimationCache[rawId] then
        local cached = DynamicAnimationCache[rawId]
        cached.lastSeen = tick()
        return cached.isAttack, cached.isProjectile, cached.regInfo
    end

    local function remember(isAttack, regInfo)
        if rawId then
            DynamicAnimationCache[rawId] = {
                isAttack = isAttack, isProjectile = false, regInfo = regInfo, lastSeen = tick(),
            }
        end
        PurgeOldestCacheEntries()
        return isAttack, false, regInfo
    end

    for i = 1, #MovementFilterKeywords do
        if string.find(lowerName, MovementFilterKeywords[i], 1, true) then
            return remember(false, nil)
        end
    end
    for i = 1, #NonAttackPatterns do
        if string.find(lowerName, NonAttackPatterns[i], 1, true) then
            return remember(false, nil)
        end
    end

    local hasAttackKeyword = false
    for i = 1, #AttackIndicatorKeywords do
        if string.find(lowerName, AttackIndicatorKeywords[i], 1, true) then
            hasAttackKeyword = true
            break
        end
    end
    if not hasAttackKeyword then
        return remember(false, nil)
    end

    local trackLength = 0
    pcall(function() trackLength = animationTrack.Length end)
    if trackLength > 0 and trackLength < 0.2 then
        return remember(false, nil)
    end

    return remember(true, {
        name = "Dynamic." .. animName,
        range = 20,
        duration = trackLength > 0 and math.min(trackLength, 1.5) or 0.5,
        isProjectile = false,
        isDynamic = true,
    })
end

-- ── enemy attack tracking ────────────────────────────────────────────────────
local EnemyTracker = {
    _enemies = {},
    _totalTracked = 0,
}

function EnemyTracker:TrackAttack(character, animTrack, isProjectile, regInfo, duration)
    local charName = character:GetFullName()
    if not self._enemies[charName] then
        self._enemies[charName] = { attacks = {}, blockIds = {}, character = character }
    end

    local enemyData = self._enemies[charName]
    local trackId = tostring(animTrack)
    if enemyData.attacks[trackId] then return end

    self._totalTracked = self._totalTracked + 1
    enemyData.attacks[trackId] = { track = animTrack, startedAt = tick() }

    local reason = "anim_" .. ((regInfo and regInfo.name) or (isProjectile and "proj" or "melee")) .. "_" .. charName
    local blockId = BlockStateMachine:Request(reason, duration)
    enemyData.blockIds[#enemyData.blockIds + 1] = blockId

    local function releaseOnce()
        if not enemyData.attacks[trackId] then return end
        enemyData.attacks[trackId] = nil
        for idx, bid in ipairs(enemyData.blockIds) do
            if bid == blockId then
                table.remove(enemyData.blockIds, idx)
                break
            end
        end
        BlockStateMachine:Release(blockId)

        local hasAttacks = false
        for _ in pairs(enemyData.attacks) do hasAttacks = true break end
        if not hasAttacks then self._enemies[charName] = nil end
    end

    -- Not tracked in Conns: Once auto-disconnects, and one entry per swing would
    -- grow that list without bound over a long session.
    animTrack.Stopped:Once(function()
        task.delay(C.releaseBuffer, releaseOnce)
    end)

    local maxTrackDuration = math.min((duration or 0.6) + 0.5, 3.0)
    task.delay(maxTrackDuration, releaseOnce)
end

function EnemyTracker:ClearCharacter(character)
    local charName = character:GetFullName()
    local enemyData = self._enemies[charName]
    if enemyData then
        for _, bid in ipairs(enemyData.blockIds) do
            BlockStateMachine:Release(bid)
        end
        self._enemies[charName] = nil
    end
end

function EnemyTracker:ClearAll()
    for name, data in pairs(self._enemies) do
        for _, bid in ipairs(data.blockIds) do
            BlockStateMachine:Release(bid)
        end
        self._enemies[name] = nil
    end
end

-- ── locked-target filter (optional) ──────────────────────────────────────────
local function GetLockedCharacter()
    local st = _G.LockState
    local t = (type(st) == "table" and st.Target) or _G.SosyLockTarget or _G.AimlockTarget
    if typeof(t) ~= "Instance" then return nil end
    if t:IsA("Model") then return t end
    return t:FindFirstAncestorOfClass("Model")
end

-- ── main animation handler ───────────────────────────────────────────────────
local function HandleAnimationPlay(character, animationTrack)
    if not Running then return end
    refreshCfg()
    if not C.enabled then return end
    if not animationTrack or not animationTrack.Animation then return end

    local localChar = GetLocalCharacter()
    if character == localChar then return end
    if character.Name == LocalPlayer.Name then return end
    if Players:GetPlayerFromCharacter(character) == LocalPlayer then return end

    if IsLocalPlayerIncapacitated() then return end
    if not IsCharacterAlive(character) then return end
    if IsCharacterIncapacitated(character) then return end

    if C.onlyLocked then
        local locked = GetLockedCharacter()
        if not locked or locked ~= character then return end
    end

    local isAttack, isProjectile, regInfo = ClassifyAnimation(animationTrack)
    if not isAttack then return end

    -- Route the hit through the SosyHUB toggle that owns it.
    local isM1 = regInfo and regInfo.isM1
    if isProjectile then
        if not C.proj then return end
    elseif isM1 then
        if not C.trail then return end
    else
        if not C.move then return end
    end

    if not C.whileAttacking and IsLocalPlayerAttacking() then return end

    local localRoot = GetLocalRoot()
    local targetRoot = character:FindFirstChild("HumanoidRootPart")
    if not localRoot or not targetRoot then return end

    local rawId = tostring(animationTrack.Animation.AnimationId):match("%d+")

    -- Range: the sliders own melee/M1 reach. Projectiles keep the registry's real
    -- reach (Kamehameha is 120 studs) but never less than the slider.
    local maxRange
    if isProjectile then
        local regRange = (regInfo and regInfo.range) or 0
        if rawId and LongRangeAttackIds[rawId] then regRange = math.max(regRange, 120) end
        maxRange = math.max(regRange, C.projRange)
    elseif isM1 then
        maxRange = C.trailDist
    else
        maxRange = C.moveRange
    end

    if GetDistanceBetween(localRoot, targetRoot) > maxRange then return end

    local dot = GetFacingDot(localRoot, targetRoot)
    local threshold
    if isProjectile then
        threshold = C.projFacing
    elseif isM1 then
        threshold = C.trailDot
    else
        threshold = C.moveFacing
    end
    if dot < threshold then return end

    if not isProjectile and IsEnemyFeinting(character) then return end

    local duration = (rawId and LongRangeTimingOverrides[rawId])
        or (regInfo and regInfo.duration)
        or C.duration
    if duration <= 0 then duration = nil end

    EnemyTracker:TrackAttack(character, animationTrack, isProjectile, regInfo, duration)
end

-- ── hit / effect reaction ────────────────────────────────────────────────────
local function StartEffectsMonitor()
    local EffectsFolder = Workspace:FindFirstChild("Effects")
    if not EffectsFolder then
        EffectsFolder = Workspace:WaitForChild("Effects", 10)
    end
    if not EffectsFolder then return end

    addConn(EffectsFolder.ChildAdded:Connect(function(child)
        if not Running then return end
        refreshCfg()
        if not (C.enabled and C.hitGlow) then return end
        if child.Name ~= "HitGlow" and child.Name ~= "BlockHit" then return end

        local localChar = GetLocalCharacter()
        local localRoot = GetLocalRoot()
        if not localChar or not localRoot then return end

        for _, descendant in ipairs(child:GetDescendants()) do
            if descendant:IsA("Weld") or descendant:IsA("WeldConstraint") then
                local p0, p1 = descendant.Part0, descendant.Part1
                if (p0 and p0:IsDescendantOf(localChar)) or (p1 and p1:IsDescendantOf(localChar)) then
                    BlockStateMachine:Request("hitglow_weld", 0.25)
                    return
                end
            end
        end

        for _, descendant in ipairs(child:GetDescendants()) do
            if descendant:IsA("BasePart") then
                if (descendant.Position - localRoot.Position).Magnitude < 8 then
                    BlockStateMachine:Request("hitglow_proximity", 0.3)
                    return
                end
            end
        end
    end))
end

-- ── physics projectile scanner ───────────────────────────────────────────────
local ProjectileNamePatterns = {
    "projectile", "bullet", "fireball", "arrow", "bolt",
    "shuriken", "orb", "sphere", "beam", "laser",
    "missile", "rocket", "bomb", "grenade", "stone",
    "ice", "rock", "fire", "wind", "water",
    "cursed", "energy", "blast", "wave", "slab",
    "kunai", "spike", "thorn", "needle", "dart",
    "cannon", "shot", "slug", "round", "pellet",
    "blood", "plasma", "void", "purple", "red",
    "blue", "black", "white", "gold", "silver",
}

local ProjScan = {
    lastScan = 0,
    interval = 0.15,
    minSpeed = 40,
    maxSpeed = 500,
    minDistance = 5,
}

local function IsLikelyProjectilePart(part)
    if not part:IsA("BasePart") then return false end
    if part.Anchored then return false end
    if part.Transparency >= 1 then return false end
    local size = part.Size
    local mag = size.Magnitude
    if mag > 12 or mag < 0.3 then return false end
    local lowerName = string.lower(part.Name)
    for _, pattern in ipairs(ProjectileNamePatterns) do
        if string.find(lowerName, pattern, 1, true) then return true end
    end
    if part.Material == Enum.Material.Neon or part.Material == Enum.Material.ForceField then
        if not part.CanCollide and size.X < 5 and size.Y < 5 and size.Z < 5 then
            return true
        end
    end
    return false
end

local function ConsiderProjectilePart(part, localPos, maxDistance, reason)
    local partPos = part.Position
    local dist = (localPos - partPos).Magnitude
    if dist <= ProjScan.minDistance or dist >= maxDistance then return end

    local velocity
    pcall(function() velocity = part.AssemblyLinearVelocity end)
    if not velocity then pcall(function() velocity = part.Velocity end) end
    if not velocity then return end

    local speed, dot = GetVelocityTowardTarget(partPos, velocity, localPos)
    if speed > ProjScan.minSpeed and speed < ProjScan.maxSpeed and dot > 0.5 then
        local timeToImpact = dist / speed
        if timeToImpact < 1.0 then
            BlockStateMachine:Request(reason, math.min(timeToImpact + 0.15, 0.8))
        end
    end
end

local function ScanPhysicsProjectiles()
    local now = tick()
    if now - ProjScan.lastScan < ProjScan.interval then return end
    ProjScan.lastScan = now

    if not Running then return end
    refreshCfg()
    if not (C.enabled and C.proj) then return end
    if IsLocalPlayerIncapacitated() then return end

    local localRoot = GetLocalRoot()
    if not localRoot then return end
    local localPos = localRoot.Position

    local EffectsFolder = Workspace:FindFirstChild("Effects")
    if not EffectsFolder then return end

    local maxDistance = math.max(C.projRange, 50)

    for _, child in ipairs(EffectsFolder:GetChildren()) do
        if child:IsA("BasePart") then
            if IsLikelyProjectilePart(child) then
                ConsiderProjectilePart(child, localPos, maxDistance, "physics_proj")
            end
        elseif child:IsA("Model") then
            for _, descendant in ipairs(child:GetDescendants()) do
                if descendant:IsA("BasePart") and IsLikelyProjectilePart(descendant) then
                    ConsiderProjectilePart(descendant, localPos, maxDistance, "physics_proj_d")
                end
            end
        end
    end
end

-- ── damage reaction ──────────────────────────────────────────────────────────
local LastHealthValue = nil
local DamageReactionCooldown = 0

local function ProcessDamageReaction()
    if not Running then return end
    refreshCfg()
    if not (C.enabled and C.damageReaction) then return end
    if IsLocalPlayerIncapacitated() then return end

    local humanoid = GetLocalHumanoid()
    if not humanoid then return end

    local currentHealth = humanoid.Health
    local now = tick()

    if LastHealthValue and currentHealth < LastHealthValue then
        local damageAmount = LastHealthValue - currentHealth
        if damageAmount > C.damageThreshold and now > DamageReactionCooldown then
            BlockStateMachine:Request("damage_reaction", C.damageHold)
            DamageReactionCooldown = now + 0.2
        end
    end
    LastHealthValue = currentHealth
end

-- ── character wiring ─────────────────────────────────────────────────────────
local function ConnectEnemyCharacter(char)
    if not Running then return end
    if not char or char == GetLocalCharacter() then return end
    -- Name check matters on our own respawn: the model lands in workspace.Characters
    -- before Player.Character is guaranteed to point at it, and wiring ourselves as
    -- an enemy would make us block on our own swings.
    if char.Name == LocalPlayer.Name then return end
    if Players:GetPlayerFromCharacter(char) == LocalPlayer then return end

    local humanoid = char:WaitForChild("Humanoid", 5)
    if not humanoid then return end

    local animator = humanoid:FindFirstChildOfClass("Animator") or humanoid:WaitForChild("Animator", 5)
    if animator then
        addConn(animator.AnimationPlayed:Connect(function(track)
            local ok, err = pcall(HandleAnimationPlay, char, track)
            if not ok and _G.SosyAutoBlockDebug then warn("[SosyHUB][AutoBlock]", err) end
        end))
    end

    addConn(char.AncestryChanged:Connect(function(_, parent)
        if not parent then EnemyTracker:ClearCharacter(char) end
    end))
    addConn(humanoid.Died:Connect(function()
        EnemyTracker:ClearCharacter(char)
    end))
end

local function SetupWorkspaceConnections()
    local CharactersFolder = GetCharactersFolder() or Workspace:WaitForChild("Characters", 10)
    if not CharactersFolder then return end

    for _, char in ipairs(CharactersFolder:GetChildren()) do
        task.spawn(ConnectEnemyCharacter, char)
    end

    addConn(CharactersFolder.ChildAdded:Connect(function(newChar)
        task.wait(0.1)
        ConnectEnemyCharacter(newChar)
    end))
    addConn(CharactersFolder.ChildRemoved:Connect(function(char)
        EnemyTracker:ClearCharacter(char)
    end))
end

local function SetupLocalRespawn()
    local function onCharacterAdded(char)
        LastHealthValue = nil
        DamageReactionCooldown = 0
        BlockStateMachine:ReleaseAll()
        EnemyTracker:ClearAll()
        task.wait(0.5)
        local humanoid = char:WaitForChild("Humanoid", 5)
        if humanoid then LastHealthValue = humanoid.Health end
        -- Enemies do not need re-wiring here: the Characters folder ChildAdded
        -- connection from SetupWorkspaceConnections survives our respawn and picks
        -- up every model that comes back.
    end

    if LocalPlayer.Character then
        task.spawn(onCharacterAdded, LocalPlayer.Character)
    end
    addConn(LocalPlayer.CharacterAdded:Connect(onCharacterAdded))
end

-- ── lifecycle ────────────────────────────────────────────────────────────────
function AB.Start()
    if Running then return end
    Running = true
    refreshCfg(true)

    task.spawn(function()
        BlockRemoteCache:Resolve()
        task.wait(1)
        if Running then SetupWorkspaceConnections() end
    end)
    task.spawn(StartEffectsMonitor)
    task.spawn(SetupLocalRespawn)

    addConn(RunService.Heartbeat:Connect(function()
        if BlockStateMachine._isBlocking then
            BlockStateMachine:_evaluateState()
        end
        pcall(ScanPhysicsProjectiles)
        pcall(ProcessDamageReaction)
        if IsLocalPlayerIncapacitated() and BlockStateMachine:IsBlocking() then
            BlockStateMachine:ReleaseAll()
        end
    end))
end

function AB.Stop()
    if not Running then return end
    Running = false
    dropConns()
    EnemyTracker:ClearAll()
    BlockStateMachine:ReleaseAll()
end

function AB.SetEnabled(on)
    _G.AutoBlockState = _G.AutoBlockState or {}
    _G.AutoBlockState.Enabled = on == true
    refreshCfg(true)
    if C.enabled then AB.Start() else AB.Stop() end
end

function AB.Sync()
    refreshCfg(true)
    if C.enabled then AB.Start() else AB.Stop() end
end

function AB.IsRunning() return Running end
function AB.IsBlocking() return BlockStateMachine:IsBlocking() end
function AB.GetStats()
    return {
        running = Running,
        blocking = BlockStateMachine._isBlocking,
        reasons = BlockStateMachine:GetActiveCount(),
        totalActivated = BlockStateMachine._totalActivated,
        totalReleased = BlockStateMachine._totalReleased,
        lastReason = BlockStateMachine._lastReason,
        remotesResolved = BlockRemoteCache._resolved,
        trackedAttacks = EnemyTracker._totalTracked,
    }
end

-- applyNative may have already flipped toggles before this chunk loaded; adopt
-- whatever state is sitting in _G.AutoBlockState right now.
task.spawn(function()
    task.wait(0.5)
    AB.Sync()
end)

return AB
end)()

-- Point the toggle callback at the real engine now that it exists.
_G.SetAutoBlock = function(on)
    _G.AutoBlockState = _G.AutoBlockState or {}
    _G.AutoBlockState.Enabled = on == true
    pcall(_G.SosyAutoBlock.SetEnabled, on == true)
end

-- ===================== Sosy native BFC / Side Dash (backup + dual-write) =====================
-- Prevent DashRemote → MovementController.Dash → Assist → DashRemote recursion (AnimationPlayed re-entrancy)
_G._SosySideDashSuppressHook = false
_G._SosySideDashFromGame = false

task.spawn(function()
    local ok, MovementController = pcall(function()
        return require(lp.PlayerScripts.Controllers.Character.MovementController)
    end)
    if not ok or not MovementController then return end
    local origDash = MovementController.Dash
    MovementController.Dash = function(self, player, char, direction)
        if player == lp then
            _G.LastGameDashDirection = direction
            _G.LastGameDashAt = tick()
            -- ONLY Left/Right side dashes — never Front/Back/Chase
            local isSide = direction == "Left" or direction == "Right"
            if not isSide then
                return origDash(self, player, char, direction)
            end
            -- Skip when assist itself fired the dash remote (avoids anim re-entrancy freeze)
            if not _G._SosySideDashSuppressHook
                and _G.OriginalSideDashEnabled then
                task.defer(function()
                    if _G._SosySideDashSuppressHook then return end
                    if _G.LastGameDashDirection ~= "Left" and _G.LastGameDashDirection ~= "Right" then
                        return
                    end
                    _G._SosySideDashFromGame = true
                    if type(_G.StartSideDashAssist) == "function" then
                        pcall(_G.StartSideDashAssist)
                    end
                    _G._SosySideDashFromGame = false
                end)
            end
        end
        return origDash(self, player, char, direction)
    end
end)

 _sosySideDash = (function() -- Side Dash Assist (RESTORED original side_dash_assist)
-- Original script body restored from side_dash_assist (2).lua
local Players=game:GetService("Players") local RunService=game:GetService("RunService")
 local UIS=game:GetService("UserInputService")
local Debris=game:GetService("Debris") local LocalPlayer=Players.LocalPlayer local Camera=workspace.CurrentCamera
local DashRemote,DashAnims,DashSound
pcall(function()
DashRemote=game.ReplicatedStorage.Knit.Knit.Services.MovementService.RE.Dash
DashAnims=game.ReplicatedStorage.Animations.Misc.Movement
DashSound=game.ReplicatedStorage.Sounds.Misc.Dash
end)
local CFG={RANGE=35,BACK_OFFSET=5.5,ARC_WIDTH=12,ARC_LIFT=2.5,DASH_TIME=0.35,BTN_SIZE=60,COOLDOWN=0.55,CAM_ASSIST=true,CHAR_LOCK=true,}
local function syncCfg()
	local s=_G.DashAssistState
	if type(s)=="table" then
		if tonumber(s.DetectionRange) then CFG.RANGE=tonumber(s.DetectionRange) end
		if tonumber(s.BehindDistance) then CFG.BACK_OFFSET=tonumber(s.BehindDistance) end
		if tonumber(s.CurveStrength) then CFG.ARC_WIDTH=tonumber(s.CurveStrength) end
		if tonumber(s.ArchHeight) then CFG.ARC_LIFT=tonumber(s.ArchHeight) end
		if tonumber(s.FlightDuration) then CFG.DASH_TIME=tonumber(s.FlightDuration) end
		if s.CameraLock~=nil then CFG.CAM_ASSIST=s.CameraLock==true end
		if s.CharLock~=nil then CFG.CHAR_LOCK=s.CharLock==true end
	end
end
pcall(function() local old=game.Players.LocalPlayer.PlayerGui:FindFirstChild("SideDashAssistGui") if old then old:Destroy() end end)
local gui=Instance.new("ScreenGui") gui.Name="SideDashAssistGui" gui.ResetOnSpawn=false gui.ZIndexBehavior=Enum.ZIndexBehavior.Sibling
local container=Instance.new("Frame") container.Name="Container" container.Size=UDim2.fromOffset(CFG.BTN_SIZE,CFG.BTN_SIZE+20) container.Position=UDim2.new(0.85,0,0.6,0) container.BackgroundTransparency=1 container.Active=true container.Parent=gui
local targetLabel=Instance.new("TextLabel") targetLabel.Size=UDim2.new(1,0,0,14) targetLabel.Position=UDim2.new(0,0,0,-16) targetLabel.BackgroundTransparency=1 targetLabel.TextColor3=Color3.new(1,1,1) targetLabel.TextSize=12 targetLabel.Font=Enum.Font.Code targetLabel.Text="" targetLabel.Parent=container
local btn=Instance.new("TextButton") btn.Size=UDim2.fromOffset(CFG.BTN_SIZE,CFG.BTN_SIZE) btn.BackgroundColor3=Color3.fromRGB(30,30,30) btn.BackgroundTransparency=0.3 btn.Text="DASH" btn.TextSize=14 btn.TextColor3=Color3.new(1,1,1) btn.Font=Enum.Font.Code btn.Parent=container
Instance.new("UICorner",btn).CornerRadius=UDim.new(0,4)
local statusLabel=Instance.new("TextLabel") statusLabel.Size=UDim2.new(1,0,0,14) statusLabel.Position=UDim2.new(0,0,1,2) statusLabel.BackgroundTransparency=1 statusLabel.TextColor3=Color3.new(0.8,0.8,0.8) statusLabel.TextSize=10 statusLabel.Font=Enum.Font.Code statusLabel.Text="None" statusLabel.Parent=container
local setBtn=Instance.new("TextButton") setBtn.Size=UDim2.fromOffset(20,20) setBtn.Position=UDim2.new(1,2,0,-10) setBtn.BackgroundColor3=Color3.fromRGB(40,40,40) setBtn.BackgroundTransparency=0.3 setBtn.Text="..." setBtn.TextColor3=Color3.new(1,1,1) setBtn.TextSize=12 setBtn.Parent=container
Instance.new("UICorner",setBtn).CornerRadius=UDim.new(0,4)
local setMenu=Instance.new("Frame") setMenu.Size=UDim2.fromOffset(180,200) setMenu.Position=UDim2.new(0.5,-90,0.5,-100) setMenu.BackgroundColor3=Color3.fromRGB(25,25,25) setMenu.BackgroundTransparency=0.1 setMenu.Visible=false setMenu.Active=true setMenu.Parent=gui
Instance.new("UICorner",setMenu).CornerRadius=UDim.new(0,4)
local title=Instance.new("TextLabel") title.Size=UDim2.new(1,0,0,24) title.BackgroundTransparency=1 title.Text="Settings" title.TextColor3=Color3.new(1,1,1) title.Font=Enum.Font.Code title.TextSize=12 title.Parent=setMenu
local layout=Instance.new("UIListLayout") layout.Padding=UDim.new(0,4) layout.Parent=setMenu
Instance.new("UIPadding",setMenu).PaddingTop=UDim.new(0,24)
local function createToggle(n,k) local f=Instance.new("Frame") f.Size=UDim2.new(1,-12,0,20) f.Position=UDim2.new(0,6,0,0) f.BackgroundTransparency=1 f.Parent=setMenu
local lbl=Instance.new("TextLabel") lbl.Size=UDim2.new(0.7,0,1,0) lbl.BackgroundTransparency=1 lbl.Text=n lbl.TextColor3=Color3.new(0.9,0.9,0.9) lbl.Font=Enum.Font.Code lbl.TextSize=10 lbl.TextXAlignment=Enum.TextXAlignment.Left lbl.Parent=f
local b=Instance.new("TextButton") b.Size=UDim2.fromOffset(30,16) b.Position=UDim2.new(1,-30,0,2) b.BackgroundColor3=CFG[k] and Color3.new(0.3,0.8,0.3) or Color3.new(0.3,0.3,0.3) b.Text="" b.Parent=f
Instance.new("UICorner",b).CornerRadius=UDim.new(0,4) b.MouseButton1Click:Connect(function() CFG[k]=not CFG[k] b.BackgroundColor3=CFG[k] and Color3.new(0.3,0.8,0.3) or Color3.new(0.3,0.3,0.3)
_G.DashAssistState=_G.DashAssistState or {}
if k=="CAM_ASSIST" then _G.DashAssistState.CameraLock=CFG[k] end
if k=="CHAR_LOCK" then _G.DashAssistState.CharLock=CFG[k] end
end) end
local function createSlider(n,k,minV,maxV,fmt) local f=Instance.new("Frame") f.Size=UDim2.new(1,-12,0,32) f.Position=UDim2.new(0,6,0,0) f.BackgroundTransparency=1 f.Parent=setMenu
local lbl=Instance.new("TextLabel") lbl.Size=UDim2.new(1,0,0,14) lbl.BackgroundTransparency=1 lbl.Text=n..": "..string.format(fmt,CFG[k]) lbl.TextColor3=Color3.new(0.9,0.9,0.9) lbl.Font=Enum.Font.Code lbl.TextSize=10 lbl.TextXAlignment=Enum.TextXAlignment.Left lbl.Parent=f
local track=Instance.new("TextButton") track.Size=UDim2.new(1,0,0,4) track.Position=UDim2.new(0,0,1,-8) track.BackgroundColor3=Color3.new(0.2,0.2,0.2) track.Text="" track.Parent=f
local fill=Instance.new("Frame") local ratio=(CFG[k]-minV)/(maxV-minV) fill.Size=UDim2.new(ratio,0,1,0) fill.BackgroundColor3=Color3.new(0.7,0.7,0.7) fill.Parent=track
local drag=false track.InputBegan:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then drag=true end end)
UIS.InputEnded:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then drag=false end end)
UIS.InputChanged:Connect(function(i) if drag and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
local p=math.clamp((i.Position.X-track.AbsolutePosition.X)/track.AbsoluteSize.X,0,1) fill.Size=UDim2.new(p,0,1,0) local val=minV+(maxV-minV)*p CFG[k]=val lbl.Text=n..": "..string.format(fmt,val)
_G.DashAssistState=_G.DashAssistState or {}
if k=="DASH_TIME" then _G.DashAssistState.FlightDuration=val end
if k=="ARC_WIDTH" then _G.DashAssistState.CurveStrength=val end
if k=="BACK_OFFSET" then _G.DashAssistState.BehindDistance=val end
end end) end
createToggle("Cam Lock","CAM_ASSIST") createToggle("Char Lock","CHAR_LOCK") createSlider("Time","DASH_TIME",0.2,0.8,"%.2f") createSlider("Width","ARC_WIDTH",0,25,"%.1f") createSlider("Offset","BACK_OFFSET",2,10,"%.1f")
setBtn.MouseButton1Click:Connect(function() setMenu.Visible=not setMenu.Visible end)
local sDrag,sStart,sPos=false,nil,nil setMenu.InputBegan:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then sDrag=true sStart=i.Position sPos=setMenu.Position end end)
UIS.InputChanged:Connect(function(i) if sDrag and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then local delta=i.Position-sStart setMenu.Position=UDim2.new(sPos.X.Scale,sPos.X.Offset+delta.X,sPos.Y.Scale,sPos.Y.Offset+delta.Y) end end)
UIS.InputEnded:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then sDrag=false end end)
local function refreshGui()
	syncCfg()
	local on=(_G.OriginalSideDashEnabled==true) or (_G.DashAssistState and _G.DashAssistState.Enabled==true)
	local mob=(_G.DashAssistState and _G.DashAssistState.MobileButtonEnabled==true)
	if on and mob then
		if not gui.Parent then gui.Parent=LocalPlayer.PlayerGui end
	else
		gui.Parent=nil
	end
end
local function quadBezier(t,p0,p1,p2) return (1-t)^2*p0+2*(1-t)*t*p1+t^2*p2 end
local function easeInOutSine(x) return -(math.cos(math.pi*x)-1)/2 end
local function getCharsFolder() return workspace:FindFirstChild("Characters") end
local function getMyChar() local f=getCharsFolder() return f and f:FindFirstChild(LocalPlayer.Name) or LocalPlayer.Character end
local function findTarget() syncCfg() local char=getMyChar() if not char then return nil,math.huge end local hrp=char:FindFirstChild("HumanoidRootPart") if not hrp then return nil,math.huge end local folder=getCharsFolder() if not folder then return nil,math.huge end
local best,bestDist=nil,CFG.RANGE for _,m in ipairs(folder:GetChildren()) do if m~=char and m:IsA("Model") then local tH=m:FindFirstChild("HumanoidRootPart") local tHum=m:FindFirstChildOfClass("Humanoid") if tH and tHum and tHum.Health>0 then
local d=(tH.Position-hrp.Position).Magnitude if d<bestDist then bestDist=d best=m end end end end return best,bestDist end
local function pickArcSide(myPos,targetCF) local toMe=(myPos-targetCF.Position).Unit local tRight=targetCF.RightVector return toMe:Dot(tRight)>0 and tRight or -tRight end
local function pickDashDirection(myPos,targetCF) local behindPos=(targetCF*CFrame.new(0,0,CFG.BACK_OFFSET)).Position local toBehind=(behindPos-myPos).Unit local camRight=Camera.CFrame.RightVector return toBehind:Dot(camRight)>0 and "Right" or "Left" end
local isDashing=false local lastDash=0 local activeTarget=nil
local function executeSideDash()
syncCfg()
if not ((_G.OriginalSideDashEnabled==true) or (_G.DashAssistState and _G.DashAssistState.Enabled==true)) then return end
if isDashing then return end if tick()-lastDash<CFG.COOLDOWN then return end
local char=getMyChar() if not char then return end local hrp=char:FindFirstChild("HumanoidRootPart") local hum=char:FindFirstChildOfClass("Humanoid") if not hrp or not hum then return end
local target,dist=findTarget() if not target then statusLabel.Text="No Tgt" return end
local tHRP=target:FindFirstChild("HumanoidRootPart") if not tHRP then return end
isDashing=true activeTarget=target lastDash=tick() statusLabel.Text="Go!"
local startPos=hrp.Position local targetCF=tHRP.CFrame local behindPos=(targetCF*CFrame.new(0,0,CFG.BACK_OFFSET)).Position
local arcDir=pickArcSide(startPos,targetCF) local dashDir=pickDashDirection(startPos,targetCF)
_G._SosySideDashSuppressHook=true
pcall(function() if DashRemote then DashRemote:FireServer(dashDir) end end)
task.delay(0.05,function() _G._SosySideDashSuppressHook=false end)
local animObj=DashAnims and DashAnims:FindFirstChild("Dash"..dashDir) if animObj and hum then local dashAnim=hum:LoadAnimation(animObj) dashAnim.Priority=Enum.AnimationPriority.Action dashAnim:Play(0.1,nil,1.8) end
pcall(function() if DashSound then local s=DashSound:Clone() s.Parent=hrp s:Play() Debris:AddItem(s,1.5) end end)
local bv=Instance.new("BodyVelocity") bv.MaxForce=Vector3.new(1e6,1e6,1e6) bv.Velocity=Vector3.zero bv.Parent=hrp Debris:AddItem(bv,CFG.DASH_TIME+0.1)
if CFG.CAM_ASSIST then RunService:BindToRenderStep("SideDashCamLock",Enum.RenderPriority.Last.Value,function()
if not isDashing or not activeTarget or not activeTarget.Parent then RunService:UnbindFromRenderStep("SideDashCamLock") return end
local liveTorso=activeTarget:FindFirstChild("Torso") or activeTarget:FindFirstChild("HumanoidRootPart") if liveTorso then Camera.CFrame=CFrame.new(Camera.CFrame.Position,liveTorso.Position) end end) end
local startTime=tick() local conn conn=RunService.RenderStepped:Connect(function(dt) local elapsed=tick()-startTime local rawT=math.clamp(elapsed/CFG.DASH_TIME,0,1) local t=easeInOutSine(rawT)
if tHRP and tHRP.Parent then behindPos=(tHRP.CFrame*CFrame.new(0,0,CFG.BACK_OFFSET)).Position end
local controlPoint=(startPos+behindPos)/2+arcDir*CFG.ARC_WIDTH+Vector3.new(0,CFG.ARC_LIFT,0) local currentPos=quadBezier(t,startPos,controlPoint,behindPos)
if CFG.CHAR_LOCK and tHRP and tHRP.Parent then hrp.CFrame=CFrame.new(currentPos,Vector3.new(tHRP.Position.X,currentPos.Y,tHRP.Position.Z)) else hrp.CFrame=CFrame.new(currentPos) end
if rawT>=1 then conn:Disconnect() if bv then bv:Destroy() end hrp.AssemblyLinearVelocity=Vector3.zero
if CFG.CHAR_LOCK and tHRP and tHRP.Parent then hrp.CFrame=CFrame.new(hrp.Position,Vector3.new(tHRP.Position.X,hrp.Position.Y,tHRP.Position.Z)) end
isDashing=false activeTarget=nil pcall(function() RunService:UnbindFromRenderStep("SideDashCamLock") end) statusLabel.Text="Rdy" end end)
task.delay(CFG.DASH_TIME+0.3,function() if conn then pcall(function() conn:Disconnect() end) end if bv then pcall(function() bv:Destroy() end) end pcall(function() RunService:UnbindFromRenderStep("SideDashCamLock") end)
if CFG.CHAR_LOCK and activeTarget and activeTarget.Parent then local liveHrp=activeTarget:FindFirstChild("HumanoidRootPart") if liveHrp and hrp and hrp.Parent then hrp.CFrame=CFrame.new(hrp.Position,Vector3.new(liveHrp.Position.X,hrp.Position.Y,liveHrp.Position.Z)) end end
isDashing=false activeTarget=nil end) end
btn.MouseButton1Click:Connect(function() executeSideDash() end)
container.InputBegan:Connect(function(input) if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then local dragOrigin=input.Position local frameOrig=container.Position local moveConn
moveConn=UIS.InputChanged:Connect(function(m) if m.UserInputType~=Enum.UserInputType.MouseMovement and m.UserInputType~=Enum.UserInputType.Touch then return end local delta=m.Position-dragOrigin if delta.Magnitude>10 then container.Position=UDim2.new(frameOrig.X.Scale,frameOrig.X.Offset+delta.X,frameOrig.Y.Scale,frameOrig.Y.Offset+delta.Y) end end)
local releaseConn releaseConn=UIS.InputEnded:Connect(function(ended) if ended==input then moveConn:Disconnect() releaseConn:Disconnect() end end) end end)
RunService.Heartbeat:Connect(function() if isDashing then return end local target,dist=findTarget() if target then targetLabel.Text=target.Name statusLabel.Text=string.format("%.0f",dist) else targetLabel.Text="" statusLabel.Text="---" end end)
LocalPlayer.CharacterAdded:Connect(function(c) c:WaitForChild("HumanoidRootPart",10) isDashing=false activeTarget=nil pcall(function() RunService:UnbindFromRenderStep("SideDashCamLock") end) end)
_G.StartSideDashAssist=executeSideDash
_G.SetSideDashAssistEnabled=function(v)
	_G.OriginalSideDashEnabled=v==true
	_G.DashAssistState=_G.DashAssistState or {}
	_G.DashAssistState.Enabled=v==true
	refreshGui()
end
_G.SetSideDashMobileButton=function(v)
	_G.DashAssistState=_G.DashAssistState or {}
	_G.DashAssistState.MobileButtonEnabled=v==true
	if v then
		_G.DashAssistState.Enabled=true
		_G.OriginalSideDashEnabled=true
	end
	refreshGui()
end
task.defer(refreshGui)
end)() -- close Side Dash IIFE

 _sosyBFC = (function() -- USER PROVIDED BFC SCRIPT --
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UIS = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

function isScriptEnabled() return _G.NewBFCEnabled or _G.BFC1Enabled end
local dashing = false
local lastBfcEnd = 0

local Settings = {
    Duration = function() return _G.DashAssistDuration or _G.NewBFCDuration or 0.35 end,
    StopDistance = function() return _G.DashAssistBehind or _G.NewBFCStopDistance or 2.0 end,
    Range = function() return _G.DashAssistRange or _G.NewBFCRange or 20 end,
    CurveStrength = function() return _G.DashAssistArc or _G.NewBFCCurveStrength or 5.5 end,
}

local LEFT_DASH_ANIM = "rbxassetid://75203303352791"
local RIGHT_DASH_ANIM = "rbxassetid://117223862448096"

-- Same blatant SwingForce-style params as Side Dash Assist
local SWING_MAX_FORCE = Vector3.new(95000, 0, 95000)
local SWING_P = 125000
local SPEED_BOOST = 1.35
local VEL_BLEND = 0.92

local AnimationTriggers = {
    ["rbxassetid://100962226150441"] = 0.19,
    ["rbxassetid://95852624447551"] = 0.19,
    ["rbxassetid://74145636023952"] = 0.19,
    ["rbxassetid://72475960800126"] = 0.20,
}

-- Base M1→BF windows are ~190–200ms. High ping needs earlier FireServer.
_G.BlackFlashState = _G.BlackFlashState or {}
if _G.BlackFlashState.AutoPingDetect == nil then _G.BlackFlashState.AutoPingDetect = true end
_G.BlackFlashState.PingCompScale = tonumber(_G.BlackFlashState.PingCompScale) or 0.55
_G.BlackFlashState.MinBFDelay = tonumber(_G.BlackFlashState.MinBFDelay) or 0.06
_G.BlackFlashState.MaxBFDelay = tonumber(_G.BlackFlashState.MaxBFDelay) or 0.28
_G.BlackFlashState.PingRefSec = tonumber(_G.BlackFlashState.PingRefSec) or 0.045

local bfPingSmooth = nil
local function sampleNetworkPing()
	local p = 0.05
	pcall(function()
		p = LocalPlayer:GetNetworkPing() or 0.05
	end)
	if typeof(p) ~= "number" or p ~= p or p < 0 then p = 0.05 end
	p = math.clamp(p, 0.01, 0.6)
	if bfPingSmooth == nil then
		bfPingSmooth = p
	else
		bfPingSmooth = bfPingSmooth * 0.78 + p * 0.22
	end
	_G._SosyBfPingSmoothed = bfPingSmooth
	return bfPingSmooth
end

local function getBfTriggerDelay(baseDelay)
	baseDelay = tonumber(baseDelay) or 0.19
	local st = _G.BlackFlashState
	if not (st and st.AutoPingDetect) then
		return baseDelay
	end
	local ping = sampleNetworkPing()
	local scale = tonumber(st.PingCompScale) or 0.55
	local ref = tonumber(st.PingRefSec) or 0.045
	-- Higher ping than ref → fire earlier (subtract more)
	local adj = baseDelay - (ping - ref) * scale
	local minD = tonumber(st.MinBFDelay) or 0.06
	local maxD = tonumber(st.MaxBFDelay) or 0.28
	return math.clamp(adj, minD, maxD)
end

_G.GetBfPingDelayInfo = function()
	local ping = sampleNetworkPing()
	local base = 0.19
	local delay = getBfTriggerDelay(base)
	return {
		auto = _G.BlackFlashState and _G.BlackFlashState.AutoPingDetect == true,
		pingMs = ping * 1000,
		delayMs = delay * 1000,
		baseMs = base * 1000,
		scale = _G.BlackFlashState and _G.BlackFlashState.PingCompScale,
	}
end

_G.SetBfAutoPingDetect = function(v)
	_G.BlackFlashState = _G.BlackFlashState or {}
	_G.BlackFlashState.AutoPingDetect = v == true
end

_G.SetBfPingCompScale = function(v)
	_G.BlackFlashState = _G.BlackFlashState or {}
	_G.BlackFlashState.PingCompScale = math.clamp(tonumber(v) or 0.55, 0.2, 1)
end

-- Keep ping sample warm (cheap)
task.spawn(function()
	while true do
		if _G.BlackFlashState and _G.BlackFlashState.AutoPingDetect then
			sampleNetworkPing()
			task.wait(0.4)
		else
			task.wait(1.5)
		end
	end
end)

local cachedRE = nil
function getActivatedRE()
    if cachedRE and cachedRE.Parent then return cachedRE end
    -- NEVER WaitForChild on the BF hot path — that freezes the client for seconds
    local function tryPath(root)
        if not root then return nil end
        local services = root:FindFirstChild("Services")
        if not services then return nil end
        local svc = services:FindFirstChild("DivergentFistService")
        if not svc then return nil end
        local reFolder = svc:FindFirstChild("RE")
        if not reFolder then return nil end
        return reFolder:FindFirstChild("Activated")
    end
    local knit = ReplicatedStorage:FindFirstChild("Knit")
    if knit then
        local re = tryPath(knit:FindFirstChild("Knit")) or tryPath(knit)
        if re then
            cachedRE = re
            return re
        end
    end
    return nil
end

function fireBlackFlash()
    if not isScriptEnabled() then return end
    local char = LocalPlayer.Character
    if not char then return end
    local moveset = char:FindFirstChild("Moveset")
    if not moveset then return end
    local move = moveset:FindFirstChild("Divergent Fist")
    if not move then return end
    local re = getActivatedRE()
    if not re then
        -- soft retry next frame without blocking
        task.defer(function()
            local re2 = getActivatedRE()
            if re2 and move.Parent then
                pcall(function() re2:FireServer(move) end)
            end
        end)
        return
    end
    pcall(function() re:FireServer(move) end)
end

function getHRP(character)
    return character and (
        character:FindFirstChild("HumanoidRootPart") or
        character:FindFirstChild("Torso") or
        character:FindFirstChild("UpperTorso")
    )
end

function isSelfStunnedOrRagdolled()
    if type(_G.IsSelfStunnedOrRagdolled) == "function" then
        return _G.IsSelfStunnedOrRagdolled()
    end
    local char = LocalPlayer.Character
    if not char then return true end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum or hum.Health <= 0 then return true end
    if char:GetAttribute("Dead") then return true end
    if hum:GetState() == Enum.HumanoidStateType.Physics then return true end
    local stunAttr = char:GetAttribute("Stun")
    if stunAttr == true or (typeof(stunAttr) == "number" and stunAttr > 0) then return true end
    if char:GetAttribute("Ragdolled") == true then return true end
    local ragdoll = char:GetAttribute("Ragdoll")
    if typeof(ragdoll) == "number" and ragdoll > 0 then return true end
    return false
end

function getNearestEnemy(maxRange)
    local myChar = LocalPlayer.Character
    local myHRP = getHRP(myChar)
    if not myHRP then return nil end

    local nearest, nearestDist = nil, maxRange

    local function consider(char)
        if not char or char == myChar or not char:IsA("Model") then return end
        local hum = char:FindFirstChildOfClass("Humanoid")
        local hrp = getHRP(char)
        if not (hum and hrp and hum.Health > 0) then return end
        if char:GetAttribute("Dead") then return end
        if hum:GetState() == Enum.HumanoidStateType.Physics then return end
        local ragdoll = char:GetAttribute("Ragdoll")
        if typeof(ragdoll) == "number" and ragdoll > 0 then return end
        local dist = (myHRP.Position - hrp.Position).Magnitude
        if dist < nearestDist and dist > 0.5 then
            nearestDist = dist
            nearest = char
        end
    end

    for _, pl in pairs(Players:GetPlayers()) do
        if pl ~= LocalPlayer and pl.Character then
            consider(pl.Character)
        end
    end

    local charFolder = workspace:FindFirstChild("Characters")
    if charFolder then
        for _, obj in ipairs(charFolder:GetChildren()) do
            consider(obj)
        end
    end

    return nearest
end

function playSideDashAnim(hum, sideSign, duration)
    local box = { track = nil }
    -- Never Play inside AnimationPlayed (re-entrancy freeze)
    task.defer(function()
        pcall(function()
            if not hum or not hum.Parent then return end
            local animator = hum:FindFirstChildOfClass("Animator") or hum
            -- Prefer already-playing game dash track
            for _, tr in ipairs(animator:GetPlayingAnimationTracks()) do
                local id = tr.Animation and tostring(tr.Animation.AnimationId or "") or ""
                if id == LEFT_DASH_ANIM or id == RIGHT_DASH_ANIM or string.find(tostring(tr.Name or ""), "Dash") then
                    box.track = tr
                    break
                end
            end
            if not box.track then
                local anim = Instance.new("Animation")
                anim.AnimationId = sideSign > 0 and RIGHT_DASH_ANIM or LEFT_DASH_ANIM
                box.track = animator:LoadAnimation(anim)
            end
            local dashTrack = box.track
            dashTrack.Priority = Enum.AnimationPriority.Action
            dashTrack.Looped = true -- hold until we arrive behind
            local len = dashTrack.Length
            local spd = 0.32
            if typeof(len) == "number" and len > 0.05 then
                spd = math.clamp(len / math.max(duration * 1.4, 0.45), 0.2, 0.55)
            end
            if not dashTrack.IsPlaying then
                dashTrack:Play(0.05, 1, spd)
            else
                dashTrack:AdjustSpeed(spd)
            end
        end)
    end)
    return box
end

local function clearBfcCameraAndAnim(hum, dashTrackBox, keepLock)
    pcall(function()
        RunService:UnbindFromRenderStep("SideDashCamLock")
    end)
    pcall(function()
        local cam = workspace.CurrentCamera
        if cam then
            cam.CameraType = Enum.CameraType.Custom
        end
    end)
    if not keepLock then
        pcall(function()
            if hum then
                hum.AutoRotate = true
            end
        end)
    end
    pcall(function()
        local dashTrack = type(dashTrackBox) == "table" and dashTrackBox.track or dashTrackBox
        if dashTrack then
            dashTrack.Looped = false
            dashTrack:Stop(0.1)
            -- only destroy tracks we created (Animation instance parentless loads are ok to destroy)
            pcall(function() dashTrack:Destroy() end)
        end
        if type(dashTrackBox) == "table" then dashTrackBox.track = nil end
    end)
end

function makeAssistSwing(myHRP, tHRP, sideSign)
    local swing = Instance.new("BodyVelocity")
    swing.Name = "SosyBFCSwing"
    swing.MaxForce = SWING_MAX_FORCE
    swing.P = SWING_P
    local flatRight = Vector3.new(myHRP.CFrame.RightVector.X, 0, myHRP.CFrame.RightVector.Z)
    if flatRight.Magnitude > 0.001 then flatRight = flatRight.Unit else flatRight = Vector3.new(1, 0, 0) end
    local sideDir = (sideSign or 1) > 0 and flatRight or -flatRight
    swing.Velocity = sideDir * 90
    swing.Parent = myHRP
    return swing
end

function startDash()
    if not isScriptEnabled() then return end
    if dashing then return end
    if tick() - lastBfcEnd < 0.12 then return end
    if isSelfStunnedOrRagdolled() then return end

    local myChar = LocalPlayer.Character
    local myHRP = getHRP(myChar)
    local hum = myChar and myChar:FindFirstChildOfClass("Humanoid")
    if not (myHRP and hum and hum.Health > 0) then return end
    if myChar:GetAttribute("Dead") then return end

    local targetChar = getNearestEnemy(Settings.Range())
    if not targetChar then return end
    local tHRP = getHRP(targetChar)
    if not tHRP or not tHRP.Parent then return end

    dashing = true
    _G._BFCTarget = targetChar

    local myStart = Vector3.new(myHRP.Position.X, 0, myHRP.Position.Z)
    local tPos = Vector3.new(tHRP.Position.X, 0, tHRP.Position.Z)
    local rightVec = Vector3.new(tHRP.CFrame.RightVector.X, 0, tHRP.CFrame.RightVector.Z)
    if rightVec.Magnitude > 0.001 then rightVec = rightVec.Unit else rightVec = Vector3.new(1, 0, 0) end

    local sideSign = (myStart - tPos):Dot(rightVec) >= 0 and 1 or -1
    local stopDist = math.max(1.0, Settings.StopDistance())
    local arcWidth = Settings.CurveStrength()
    local duration = math.max(0.18, Settings.Duration() * 0.92)

    -- Capture how far from behind we started so close-range dashes don't cancel early
    local lookVec0 = Vector3.new(tHRP.CFrame.LookVector.X, 0, tHRP.CFrame.LookVector.Z)
    if lookVec0.Magnitude > 0.001 then lookVec0 = lookVec0.Unit else lookVec0 = Vector3.new(0, 0, -1) end
    local startBehindDist = (myStart - (tPos - lookVec0 * stopDist)).Magnitude

    -- Body faces BFC target (never camera)
    pcall(function()
        hum.AutoRotate = false
    end)
    local function faceBodyAtTarget()
        if not (myHRP and myHRP.Parent and tHRP and tHRP.Parent) then return end
        local lookAt = Vector3.new(tHRP.Position.X, myHRP.Position.Y, tHRP.Position.Z)
        if (lookAt - myHRP.Position).Magnitude < 0.05 then return end
        myHRP.CFrame = CFrame.new(myHRP.Position, lookAt)
    end
    faceBodyAtTarget()

    -- Instant SwingForce-style assist (same as side dash)
    local ownedSwing = false
    local swing = myHRP:FindFirstChild("SwingForce")
    if not (swing and swing:IsA("BodyVelocity")) then
        swing = makeAssistSwing(myHRP, tHRP, sideSign)
        ownedSwing = true
    end

    local baseSpeed = math.max(swing.Velocity.Magnitude, 70)
    local dashTrackBox = playSideDashAnim(hum, sideSign, duration)
    local startTime = tick()

    local conn
    conn = RunService.Heartbeat:Connect(function()
        local function finish()
            pcall(function() conn:Disconnect() end)
            if ownedSwing and swing and swing.Parent and swing.Name == "SosyBFCSwing" then
                pcall(function() swing:Destroy() end)
            end
            -- final body face, then hold lock for ~0.4s so BF fires while still locked on target
            pcall(faceBodyAtTarget)
            clearBfcCameraAndAnim(hum, dashTrackBox, true)
            _G._BFCTarget = nil
            dashing = false
            lastBfcEnd = tick()
            -- Keep body locked on target after dash ends so BF fires while facing them
            local _postEnd = tick() + 0.4
            local _postConn
            _postConn = RunService.Heartbeat:Connect(function()
                if dashing or tick() >= _postEnd
                    or not (tHRP and tHRP.Parent and myHRP and myHRP.Parent and hum and hum.Parent) then
                    if not dashing then
                        pcall(function() if hum then hum.AutoRotate = true end end)
                    end
                    pcall(function() _postConn:Disconnect() end)
                    return
                end
                pcall(faceBodyAtTarget)
            end)
        end

        if not myChar.Parent then finish() return end
        myHRP = getHRP(myChar)
        if not myHRP then finish() return end
        if not hum.Parent or hum.Health <= 0 then finish() return end
        -- Do NOT abort on micro-Stun during dash — that freezes the combo and blocks Black Flash
        if myChar:GetAttribute("Dead") then finish() return end
        local ragdoll = myChar:GetAttribute("Ragdoll")
        if typeof(ragdoll) == "number" and ragdoll > 0 then finish() return end
        if not targetChar.Parent then finish() return end
        tHRP = getHRP(targetChar)
        if not tHRP then finish() return end

        local liveSwing = myHRP:FindFirstChild("SwingForce")
        if liveSwing and liveSwing:IsA("BodyVelocity") then
            if ownedSwing and swing and swing.Parent and swing ~= liveSwing then
                pcall(function() swing:Destroy() end)
                ownedSwing = false
            end
            swing = liveSwing
        elseif not swing or not swing.Parent then
            finish()
            return
        end

        local elapsed = tick() - startTime
        if elapsed > duration + 0.35 then
            finish()
            return
        end

        local liveRight = Vector3.new(tHRP.CFrame.RightVector.X, 0, tHRP.CFrame.RightVector.Z)
        if liveRight.Magnitude > 0.001 then liveRight = liveRight.Unit else liveRight = rightVec end
        local liveLook = Vector3.new(tHRP.CFrame.LookVector.X, 0, tHRP.CFrame.LookVector.Z)
        if liveLook.Magnitude > 0.001 then liveLook = liveLook.Unit else liveLook = Vector3.new(0, 0, -1) end

        local liveTarget = Vector3.new(tHRP.Position.X, 0, tHRP.Position.Z)
        local liveSide = liveTarget + liveRight * (arcWidth * sideSign)
        local liveBehind = liveTarget - liveLook * stopDist

        local myFlat = Vector3.new(myHRP.Position.X, 0, myHRP.Position.Z)
        local alpha = math.clamp(elapsed / math.max(duration, 0.01), 0, 1)

        local goal
        if alpha < 0.28 then
            local a = alpha / 0.28
            local eased = 1 - (1 - a) ^ 3
            goal = myStart:Lerp(liveSide, 0.55 + eased * 0.45)
        else
            local a = (alpha - 0.28) / 0.72
            local eased = 1 - (1 - a) ^ 2
            goal = liveSide:Lerp(liveBehind, eased)
            local lock = math.clamp((a - 0.08) / 0.55, 0, 1)
            goal = goal:Lerp(liveBehind, lock * lock)
        end

        local toGoal = goal - myFlat
        local distBehind = (myFlat - liveBehind).Magnitude

        local speed
        if not ownedSwing then
            speed = math.max(swing.Velocity.Magnitude, 28) * SPEED_BOOST
        else
            speed = baseSpeed * SPEED_BOOST * math.max(0.28, 1 - alpha * 0.7)
        end
        speed = math.clamp(speed, 38, 140)

        if toGoal.Magnitude > 0.06 then
            local desired = toGoal.Unit * speed
            local cur = Vector3.new(swing.Velocity.X, 0, swing.Velocity.Z)
            local blended = cur:Lerp(desired, VEL_BLEND)
            swing.MaxForce = SWING_MAX_FORCE
            swing.P = SWING_P
            swing.Velocity = Vector3.new(blended.X, 0, blended.Z)
        end

        -- keep body locked on BFC enemy (camera untouched)
        faceBodyAtTarget()

        -- dot-product check: how far the player is geometrically behind the target
        local behindDepth = (myFlat - liveTarget):Dot(-liveLook)
        local cancelAlpha = startBehindDist > stopDist + 0.2 and 0.22 or 0.65
        if behindDepth >= stopDist and alpha >= cancelAlpha then
            finish()
            return
        end

        if alpha >= 1 and distBehind <= 4 then
            finish()
        end
    end)
end

_G.StartBFCDash = startDash

-- Dedicated BF mobile button (not Side Dash)
do
	local bfGui, bfBtn
	local function destroyBfGui()
		if bfGui then pcall(function() bfGui:Destroy() end) end
		bfGui, bfBtn = nil, nil
	end
	local function ensureBfGui()
		if bfGui and bfGui.Parent then return end
		destroyBfGui()
		bfGui = Instance.new("ScreenGui")
		bfGui.Name = "SosyBFCMobileGui"
		bfGui.ResetOnSpawn = false
		bfGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
		bfGui.Parent = LocalPlayer:FindFirstChildOfClass("PlayerGui") or LocalPlayer:WaitForChild("PlayerGui")
		local box = Instance.new("Frame")
		box.Size = UDim2.fromOffset(64, 64)
		box.Position = UDim2.new(0.78, 0, 0.72, 0)
		box.BackgroundTransparency = 1
		box.Active = true
		box.Parent = bfGui
		bfBtn = Instance.new("TextButton")
		bfBtn.Size = UDim2.fromScale(1, 1)
		bfBtn.BackgroundColor3 = Color3.fromRGB(28, 28, 32)
		bfBtn.BackgroundTransparency = 0.2
		bfBtn.Text = "BF"
		bfBtn.TextColor3 = Color3.new(1, 1, 1)
		bfBtn.Font = Enum.Font.GothamBold
		bfBtn.TextSize = 16
		bfBtn.Parent = box
		Instance.new("UICorner", bfBtn).CornerRadius = UDim.new(1, 0)
		local bfDragged = false
		local dragOrigin, frameOrig, moveConn, releaseConn
		local function beginBfDrag(input)
			if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
			bfDragged = false
			dragOrigin = input.Position
			frameOrig = box.Position
			if moveConn then moveConn:Disconnect() end
			if releaseConn then releaseConn:Disconnect() end
			moveConn = UIS.InputChanged:Connect(function(m)
				if m.UserInputType ~= Enum.UserInputType.MouseMovement and m.UserInputType ~= Enum.UserInputType.Touch then return end
				local delta = m.Position - dragOrigin
				if delta.Magnitude > 6 then
					bfDragged = true
					box.Position = UDim2.new(frameOrig.X.Scale, frameOrig.X.Offset + delta.X, frameOrig.Y.Scale, frameOrig.Y.Offset + delta.Y)
				end
			end)
			releaseConn = UIS.InputEnded:Connect(function(ended)
				if ended == input then
					if moveConn then moveConn:Disconnect() end
					if releaseConn then releaseConn:Disconnect() end
				end
			end)
		end
		bfBtn.InputBegan:Connect(beginBfDrag)
		box.InputBegan:Connect(beginBfDrag)
		bfBtn.MouseButton1Click:Connect(function()
			if bfDragged then bfDragged = false return end
			if type(_G.StartBFCDash) == "function" then pcall(_G.StartBFCDash) end
		end)
	end
	_G.SetBFCMobileButton = function(v)
		_G.BFCMobileButtonEnabled = v == true
		if v then ensureBfGui() else destroyBfGui() end
	end
	task.defer(function()
		if _G.BFCMobileButtonEnabled == true then ensureBfGui() end
	end)
end

local function resolveBfcKey()
	local binds = _G.SosyKeybinds
	if type(binds) == "table" then
		for _, n in ipairs({ "Black Flash", "Black Flash Chain", "Enable Black Flash", "BFC" }) do
			local k = binds[n]
			if type(k) == "string" and k ~= "" and Enum.KeyCode[k] then
				return Enum.KeyCode[k]
			end
		end
	end
	return Enum.KeyCode.Three
end

UIS.InputBegan:Connect(function(input, processed)
    if processed then return end
    if not isScriptEnabled() then return end
    if input.KeyCode == resolveBfcKey() then
        startDash()
    end
end)

function setupCharacter(character)
    local humanoid = character:WaitForChild("Humanoid", 5)
    local animator = humanoid and humanoid:WaitForChild("Animator", 5)
    if not animator then return end

    -- Warm BF remote cache so M1→BF never WaitForChild-hitches
    task.defer(function()
        pcall(getActivatedRE)
    end)

    animator.AnimationPlayed:Connect(function(track)
        if not isScriptEnabled() then return end
        local anim = track and track.Animation
        local animId = anim and anim.AnimationId
        if not animId then return end
        -- ignore our own side-dash tracks
        if animId == LEFT_DASH_ANIM or animId == RIGHT_DASH_ANIM then return end
        local delayTime = AnimationTriggers[animId]
        if not delayTime then return end
        delayTime = getBfTriggerDelay(delayTime)
        -- never do work synchronously inside AnimationPlayed
        task.defer(function()
            task.delay(delayTime, function()
                if not (humanoid.Health > 0 and isScriptEnabled()) then return end
                fireBlackFlash()
            end)
        end)
    end)
end

if LocalPlayer.Character then setupCharacter(LocalPlayer.Character) end
LocalPlayer.CharacterAdded:Connect(function(char)
    dashing = false
    lastBfcEnd = 0
    setupCharacter(char)
end)
end)() -- close BFC IIFE
-- ===================== /Sosy native BFC / Side Dash =====================
-- ===================== /Ryu + BFC =====================

-- ===================== Specials natives (Auto QTE + Yuki Instant Charge) =====================
do
	ensure("SpecialsState", {
		HiguQTE = false,
		YukiInstantCharge = false,
		TodoBringRange = 100,
		TodoBringDelay = 0.18,
	})

	--[[
		Todo Swap Bring (Boogie Woogie bring):
		1) Remember where YOU are (bring destination)
		2) Teleport onto the target
		3) Clap = TodoService RightActivated(target)
		4) Teleport back — target is swapped toward that spot
		Requires Moveset == "Todo".
	]]
	function _G.TodoSwapBring(explicitTarget)
		local chars = workspace:FindFirstChild("Characters")
		local myChar = (chars and chars:FindFirstChild(lp.Name)) or lp.Character
		if not myChar then return false, "no character" end
		local myHRP = myChar:FindFirstChild("HumanoidRootPart") or myChar.PrimaryPart
		if not myHRP then return false, "no hrp" end

		local moveset = tostring(myChar:GetAttribute("Moveset") or "")
		if moveset ~= "Todo" then
			return false, "need Todo moveset (now: " .. moveset .. ")"
		end

		local range = tonumber(_G.SpecialsState and _G.SpecialsState.TodoBringRange) or 100
		local delaySec = tonumber(_G.SpecialsState and _G.SpecialsState.TodoBringDelay) or 0.18

		local function asChar(inst)
			if not inst then return nil end
			if inst:IsA("Model") then return inst end
			if inst:IsA("BasePart") then return inst.Parent end
			return nil
		end

		local targetChar = asChar(explicitTarget)
		if not targetChar and type(_G.GetLockedTarget) == "function" then
			pcall(function() targetChar = asChar(_G.GetLockedTarget()) end)
		end
		if not targetChar then
			local best, bestD = nil, range
			local folder = chars or workspace
			for _, ch in ipairs(folder:GetChildren()) do
				if ch ~= myChar and ch:IsA("Model") then
					local hum = ch:FindFirstChildOfClass("Humanoid")
					local hrp = ch:FindFirstChild("HumanoidRootPart")
					if hum and hrp and hum.Health > 0 then
						local d = (hrp.Position - myHRP.Position).Magnitude
						if d <= bestD then
							bestD = d
							best = ch
						end
					end
				end
			end
			targetChar = best
		end
		if not targetChar or targetChar == myChar then return false, "no target" end
		local tHRP = targetChar:FindFirstChild("HumanoidRootPart") or targetChar.PrimaryPart
		if not tHRP then return false, "target has no hrp" end

		local bringCF = myHRP.CFrame
		-- 1) go to them
		pcall(function()
			myHRP.AssemblyLinearVelocity = Vector3.zero
			myHRP.CFrame = tHRP.CFrame * CFrame.new(0, 0, 1.25)
		end)
		task.wait(0.05)

		-- 2) clap / Boogie Woogie (same as ToolController RightActivated for Todo)
		local fired = false
		pcall(function()
			local re = RS.Knit.Knit.Services.TodoService.RE.RightActivated
			re:FireServer(targetChar)
			fired = true
		end)
		if not fired then
			pcall(function()
				myHRP.CFrame = bringCF
			end)
			return false, "clap remote failed"
		end

		task.wait(math.clamp(delaySec, 0.05, 0.6))

		-- 3) return to bring destination
		pcall(function()
			if myHRP and myHRP.Parent then
				myHRP.AssemblyLinearVelocity = Vector3.zero
				myHRP.CFrame = bringCF
			end
		end)
		return true, targetChar.Name
	end

	local function stopAutoQTE()
		local st = _G.SpecialsState
		if not st then return end
		if st.AutoQTEConn then
			pcall(function() st.AutoQTEConn:Disconnect() end)
			st.AutoQTEConn = nil
		end
	end

	function _G.SetAutoQTE(v)
		ensure("SpecialsState", { HiguQTE = false })
		_G.SpecialsState.HiguQTE = v == true
		stopAutoQTE()
		if not _G.SpecialsState.HiguQTE then return end
		local lastTick, lastPressed = 0, 0
		_G.SpecialsState.AutoQTEConn = RunService.Heartbeat:Connect(function()
			if not (_G.SpecialsState and _G.SpecialsState.HiguQTE) then return end
			if tick() - lastTick < 0.1 then return end
			lastTick = tick()
			local playerGui = lp:FindFirstChild("PlayerGui")
			if not playerGui then return end
			local qte = playerGui:FindFirstChild("QTE")
			if not qte then return end
			local pc = qte:FindFirstChild("QTE_PC")
			if not (pc and pc:IsA("TextLabel")) then return end
			local keyStr = string.match(string.upper(tostring(pc.Text or "")), "[A-Z]")
			if not keyStr or tick() - lastPressed <= 0.2 then return end
			lastPressed = tick()
			local keyCode = Enum.KeyCode[keyStr]
			if not keyCode then return end
			task.spawn(function()
				local vim = game:GetService("VirtualInputManager")
				pcall(function()
					vim:SendKeyEvent(true, keyCode, false, game)
					task.wait(0.03)
					vim:SendKeyEvent(false, keyCode, false, game)
				end)
			end)
		end)
	end

	local yukiChargeConn = nil
	function _G.SetYukiInstantCharge(v)
		ensure("SpecialsState", { YukiInstantCharge = false })
		_G.SpecialsState.YukiInstantCharge = v == true
		if yukiChargeConn then
			pcall(function() yukiChargeConn:Disconnect() end)
			yukiChargeConn = nil
		end
		if not _G.SpecialsState.YukiInstantCharge then return end
		local yukiAcc = 0
		yukiChargeConn = RunService.Heartbeat:Connect(function(dt)
			if not (_G.SpecialsState and _G.SpecialsState.YukiInstantCharge) then return end
			yukiAcc = yukiAcc + dt
			if yukiAcc < 0.1 then return end
			yukiAcc = 0
			local char = lp.Character
			if not char then return end
			pcall(function()
				if char:GetAttribute("Charging") then
					char:SetAttribute("Charge", 100)
				end
				local chargeAttr = char:GetAttribute("Charge")
				if type(chargeAttr) == "number" and chargeAttr < 100 and char:GetAttribute("Charging") then
					char:SetAttribute("Charge", 100)
				end
			end)
			-- Avoid full GetDescendants — only scan known meter containers
			pcall(function()
				local info = char:FindFirstChild("Info")
				local roots = { info, char:FindFirstChild("Moveset"), char }
				for _, root in ipairs(roots) do
					if root then
						for _, d in ipairs(root:GetChildren()) do
							if d:IsA("NumberValue") or d:IsA("IntValue") then
								local maxV = d:GetAttribute("Max")
								if type(maxV) == "number" and maxV > 0 and d.Value < maxV then
									local n = string.lower(d.Name)
									if n:find("charge") or n:find("meter") or n:find("stack") or d:GetAttribute("Cost") ~= nil then
										d.Value = maxV
									end
								end
							end
						end
					end
				end
			end)
		end)
	end

	-- Rebrand any SEDSE/TBO AI chrome to SosyHUB
	local function rebrandTboText(inst)
		if not inst then return end
		pcall(function()
			if inst:IsA("TextLabel") or inst:IsA("TextButton") or inst:IsA("TextBox") then
				local t = tostring(inst.Text or "")
				if t ~= "" and (string.find(t, "TBO", 1, true) or string.find(string.lower(t), "tbo assistant")) then
					inst.Text = t
						:gsub("TBO Assistant", "SosyHUB")
						:gsub("TBO assistant", "SosyHUB")
						:gsub("tbo assistant", "SosyHUB")
						:gsub("I am TBO", "I am SosyHUB")
						:gsub("I'm TBO", "I'm SosyHUB")
						:gsub("TBO", "SosyHUB")
				end
			end
			if inst.Name and string.find(tostring(inst.Name), "TBO", 1, true) then
				inst.Name = tostring(inst.Name):gsub("TBO", "SosyHUB")
			end
		end)
	end

	task.spawn(function()
		-- Only touch SosyHUB GUIs — full PlayerGui:GetDescendants every 0.75s was freezing the game
		if type(_G._SosyTboScanCleanup) == "function" then
			pcall(_G._SosyTboScanCleanup)
		end
		local conns = {}
		_G._SosyTboScanCleanup = function()
			for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
			table.clear(conns)
		end
		local function sosyRoots()
			local roots = {}
			local parents = {}
			pcall(function() if gethui then table.insert(parents, gethui()) end end)
			table.insert(parents, game:GetService("CoreGui"))
			local pg = lp:FindFirstChild("PlayerGui")
			if pg then table.insert(parents, pg) end
			for _, p in ipairs(parents) do
				for _, c in ipairs(p:GetChildren()) do
					if type(c.Name) == "string" and string.sub(c.Name, 1, 7) == "SosyHUB" then
						table.insert(roots, c)
					end
				end
			end
			return roots
		end
		local function scanSosy()
			for _, root in ipairs(sosyRoots()) do
				for _, d in ipairs(root:GetDescendants()) do
					rebrandTboText(d)
				end
			end
		end
		scanSosy()
		local pg = lp:FindFirstChild("PlayerGui") or lp:WaitForChild("PlayerGui", 30)
		if pg then
			table.insert(conns, pg.ChildAdded:Connect(function(child)
				if child and string.sub(tostring(child.Name), 1, 7) == "SosyHUB" then
					task.defer(function()
						for _, d in ipairs(child:GetDescendants()) do rebrandTboText(d) end
						table.insert(conns, child.DescendantAdded:Connect(function(d)
							task.defer(rebrandTboText, d)
						end))
					end)
				end
			end))
		end
		pcall(function()
			local hui = gethui and gethui()
			if hui then
				table.insert(conns, hui.ChildAdded:Connect(function(child)
					if child and string.sub(tostring(child.Name), 1, 7) == "SosyHUB" then
						task.defer(function()
							for _, d in ipairs(child:GetDescendants()) do rebrandTboText(d) end
							table.insert(conns, child.DescendantAdded:Connect(function(d)
								task.defer(rebrandTboText, d)
							end))
						end)
					end
				end))
			end
		end)
	end)

	_G.SosyRebrandAIText = function(text)
		text = tostring(text or "")
		text = text
			:gsub("TBO Assistant", "SosyHUB")
			:gsub("TBO assistant", "SosyHUB")
			:gsub("tbo assistant", "SosyHUB")
			:gsub("I am TBO%s*assistant", "I am SosyHUB")
			:gsub("I'm TBO%s*assistant", "I'm SosyHUB")
			:gsub("I am the TBO", "I am SosyHUB")
			:gsub("TBO", "SosyHUB")
		return text
	end
end
-- ===================== /Specials natives =====================

-- ===================== Infinite Dash (Sedse dump: tick hook) =====================
-- Sedse bypasses dash cooldown by advancing hooked tick() — not stun clearing alone.
do
	ensure("MiscState", {
		InfiniteDashActive = false,
		AutoBuySoda = false,
		SodaBuyHpPct = 40,
	})
	_G.MiscState.OriginalTick = _G.MiscState.OriginalTick or tick

	function _G.SetInfiniteDash(v)
		ensure("MiscState", { InfiniteDashActive = false })
		local on = v == true
		_G.MiscState.InfiniteDashActive = on
		_G.MiscState.OriginalTick = _G.MiscState.OriginalTick or tick

		if type(hookfunction) ~= "function" then
			warn("[SosyHUB] Infinite Dash needs hookfunction (executor)")
			return
		end

		if on then
			_G.MiscState.FakeTime = _G.MiscState.OriginalTick()
			pcall(function()
				hookfunction(tick, function()
					_G.MiscState.FakeTime = (_G.MiscState.FakeTime or 0) + 100
					return _G.MiscState.FakeTime
				end)
			end)
		else
			pcall(function()
				hookfunction(tick, _G.MiscState.OriginalTick)
			end)
		end
	end

	task.defer(function()
		if _G.MiscState and _G.MiscState.InfiniteDashActive then
			_G.SetInfiniteDash(true)
		end
	end)
end
-- ===================== /Infinite Dash =====================

-- ===================== Infinite Parkour =====================
-- Wraps MovementController.Parkour so the counter+timestamp are zeroed
-- on EVERY call, and also clears any character attributes that gate parkour.
do
	ensure("MiscState", { InfiniteParkour = false })

	function _G.SetInfiniteParkour(on)
		ensure("MiscState", { InfiniteParkour = false })
		_G.MiscState.InfiniteParkour = on == true
	end

	if not _G._SosyInfiniteParkourLoop then
		_G._SosyInfiniteParkourLoop = true

		local RS      = game:GetService("RunService")
		local Players = game:GetService("Players")
		local lp      = Players.LocalPlayer

		-- Resolve the best available upvalue getter/setter
		local function getUvGS()
			local g = (debug and type(debug) == "table" and debug.getupvalue) or
			          (type(getupvalue) == "function" and getupvalue) or nil
			local s = (debug and type(debug) == "table" and debug.setupvalue) or
			          (type(setupvalue) == "function" and setupvalue) or nil
			return g, s
		end

		-- Given a function, scan and zero: small-number upvalues (counters) and
		-- large-number upvalues (timestamps). Named u13/u14 are priority targets.
		local function resetParkourUpvalues(fn)
			local g, s = getUvGS()
			if not g or not s then return end
			local counterIdx, timeIdx = nil, nil
			for i = 1, 30 do
				local ok, nameOrVal, val2 = pcall(function()
					if debug and debug.getupvalue then
						return debug.getupvalue(fn, i)
					end
					return nil, g(fn, i)
				end)
				if not ok then break end
				local name = (type(nameOrVal) == "string") and nameOrVal or nil
				local v    = name and val2 or nameOrVal
				if v == nil and name == nil then break end
				if name == "u13" then counterIdx = i end
				if name == "u14" then timeIdx    = i end
				if not name then
					if type(v) == "number" and v >= 0 and v <= 10 and not counterIdx then
						counterIdx = i
					elseif type(v) == "number" and counterIdx and not timeIdx then
						timeIdx = i
					end
				end
			end
			if counterIdx then pcall(s, fn, counterIdx, 0)    end
			if timeIdx    then pcall(s, fn, timeIdx,    -1e9) end
			-- Brute-force fallback: zero every small-integer upvalue
			if not counterIdx then
				for i = 1, 20 do
					local ok2, v = pcall(g, fn, i)
					if not ok2 then break end
					if type(v) == "number" and v > 0 and v <= 10 then
						pcall(s, fn, i, 0)
					elseif type(v) == "number" and v > 1e6 then
						pcall(s, fn, i, -1e9)
					end
				end
			end
		end

		-- Wrap MovementController.Parkour once so resets happen at call-time
		task.defer(function()
			pcall(function()
				local ctrl
				-- Path 1: direct require
				local ok1, r1 = pcall(require, lp.PlayerScripts.Controllers.Character.MovementController)
				if ok1 and type(r1) == "table" then ctrl = r1 end
				-- Path 2: Knit
				if not ctrl then
					pcall(function()
						local Knit = require(game:GetService("ReplicatedStorage").Knit.Knit)
						ctrl = Knit.GetController("MovementController")
					end)
				end
				if not ctrl then return end

				-- Replace Parkour with a counter-resetting wrapper
				local names = { "Parkour", "WallRun", "Wallrun", "wallRun" }
				for _, fname in ipairs(names) do
					if type(ctrl[fname]) == "function" then
						local origFn = ctrl[fname]
						ctrl[fname] = function(self, ...)
							if _G.MiscState and _G.MiscState.InfiniteParkour then
								resetParkourUpvalues(origFn)
							end
							return origFn(self, ...)
						end
					end
				end
			end)
		end)

		-- Attribute/heartbeat fallback (handles games that gate via char attributes)
		RS.Heartbeat:Connect(function()
			if not (_G.MiscState and _G.MiscState.InfiniteParkour) then return end
			pcall(function()
				local char = lp.Character
				if not char then return end
				char:SetAttribute("Parkour", false)
				if char:GetAttribute("WallRuns")     then char:SetAttribute("WallRuns",     0) end
				if char:GetAttribute("WallRunCount") then char:SetAttribute("WallRunCount", 0) end
			end)
		end)
	end

	task.defer(function()
		if _G.MiscState and _G.MiscState.InfiniteParkour then
			pcall(_G.SetInfiniteParkour, true)
		end
	end)
end
-- ===================== /Infinite Parkour =====================

-- ===================== Anti Head Bounce =====================
-- Detects whether we are standing on another player's character part.
-- If yes: cap both Y (prevents sky-launch) and X/Z (prevents fling).
-- If no (map/terrain): do NOT interfere so legitimate parkour moves are unaffected.
do
	local RS      = game:GetService("RunService")
	local Players = game:GetService("Players")
	local lp      = Players.LocalPlayer
	local YCAP      = 50   -- above any normal jump (~50 u/s); kills head-boost
	local XCAP_HEAD = 60   -- lateral speed cap when on a head to stop fling

	if not _G._SosyAntiHeadBounce then
		_G._SosyAntiHeadBounce = true
		local castParams = RaycastParams.new()
		castParams.FilterType = Enum.RaycastFilterType.Exclude

		RS.Heartbeat:Connect(function()
			local char = lp.Character
			if not char then return end
			local hrp = char:FindFirstChild("HumanoidRootPart")
			if not hrp then return end
			local vel = hrp.AssemblyLinearVelocity
			if vel.Y <= YCAP then return end

			-- Exclude our own character; hit everything else (map, terrain, players)
			castParams.FilterDescendantsInstances = {char}
			local hit = workspace:Raycast(hrp.Position, Vector3.new(0, -4, 0), castParams)
			if not hit then return end  -- nothing below → air, leave physics alone

			-- Determine whether what's below us is a player's character part
			local inst = hit.Instance
			if not inst then return end
			local parent = inst.Parent
			local isPlayerChar = false
			if parent then
				if parent:FindFirstChildOfClass("Humanoid") then
					isPlayerChar = true
				elseif parent.Parent and parent.Parent:FindFirstChildOfClass("Humanoid") then
					isPlayerChar = true
				end
			end

			if isPlayerChar then
				-- Cap Y and dampen lateral fling velocity
				local cx = math.clamp(vel.X, -XCAP_HEAD, XCAP_HEAD)
				local cz = math.clamp(vel.Z, -XCAP_HEAD, XCAP_HEAD)
				hrp.AssemblyLinearVelocity = Vector3.new(cx, YCAP, cz)
			end
			-- If not a player (map/terrain), do nothing → parkour unaffected
		end)
	end
end
-- ===================== /Anti Head Bounce =====================

-- ===================== Auto Buy Soda When Low HP =====================
do
	local RS      = game:GetService("RunService")
	local Players = game:GetService("Players")
	local lp      = Players.LocalPlayer
	ensure("MiscState", { AutoBuySoda = false, SodaBuyHpPct = 40 })
	local sodaConn = nil
	local lastBuy = 0

	local function realNow()
		local ot = _G.MiscState and _G.MiscState.OriginalTick
		if type(ot) == "function" then
			local ok, t = pcall(ot)
			if ok and type(t) == "number" then return t end
		end
		-- fallback: don't use hooked tick while Infinite Dash is on
		return os.clock()
	end

	local function getPurchaseSoda()
		local ok, re = pcall(function()
			return game.ReplicatedStorage.Knit.Knit.Services.ShopService.RE.PurchaseSoda
		end)
		if ok and re then return re end
		return nil
	end

	function _G.SetAutoBuySoda(v)
		ensure("MiscState", { AutoBuySoda = false, SodaBuyHpPct = 40 })
		_G.MiscState.AutoBuySoda = v == true
		if sodaConn then
			pcall(function() sodaConn:Disconnect() end)
			sodaConn = nil
		end
		if not _G.MiscState.AutoBuySoda then return end
		sodaConn = RS.Heartbeat:Connect(function()
			if not (_G.MiscState and _G.MiscState.AutoBuySoda) then return end
			local now = realNow()
			if now - lastBuy < 2.5 then return end
			local char = lp.Character
			local hum = char and char:FindFirstChildOfClass("Humanoid")
			if not hum or hum.Health <= 0 then return end
			local maxH = hum.MaxHealth
			if typeof(maxH) ~= "number" or maxH < 1 then maxH = 100 end
			local pct = tonumber(_G.MiscState.SodaBuyHpPct) or 40
			if (hum.Health / maxH) * 100 > pct then return end
			local re = getPurchaseSoda()
			if not re then return end
			lastBuy = now
			pcall(function() re:FireServer() end)
			pcall(function() re:Fire() end)
		end)
	end

	task.defer(function()
		if _G.MiscState and _G.MiscState.AutoBuySoda then
			_G.SetAutoBuySoda(true)
		end
	end)
end
-- ===================== /Auto Buy Soda =====================

-- ===================== 0.2 Domain Farm (Invisible → punch → G+R spam → Rapid TP) =====================
do
	ensure("SpecialsState", {
		Domain02Farm = false,
		Domain02Phase = "idle",
	})
	local farmConn = nil
	local casting = false
	local domainLatched = false
	local lastPunch = 0
	local lastCast = 0
	local vim = game:GetService("VirtualInputManager")

	local function realNow()
		local ot = _G.MiscState and _G.MiscState.OriginalTick
		if type(ot) == "function" then
			local ok, t = pcall(ot)
			if ok and type(t) == "number" then return t end
		end
		return os.clock()
	end

	local function tapKey(code, hold)
		pcall(function()
			vim:SendKeyEvent(true, code, false, game)
		end)
		task.wait(hold or 0.04)
		pcall(function()
			vim:SendKeyEvent(false, code, false, game)
		end)
	end

	local function fireMelee()
		pcall(function()
			local melee = game.ReplicatedStorage.Keybind.Combat.Melee
			if melee and typeof(getconnections) == "function" and melee.Pressed then
				for _, c in ipairs(getconnections(melee.Pressed)) do
					pcall(function()
						if c.Fire then c:Fire() elseif c.Function then c.Function() end
					end)
				end
				return
			end
		end)
		pcall(function()
			vim:SendMouseButtonEvent(0, 0, 0, true, game, 0)
			task.wait(0.03)
			vim:SendMouseButtonEvent(0, 0, 0, false, game, 0)
		end)
	end

	local function setInvisible(on)
		_G.MiscState = _G.MiscState or {}
		_G.MiscState.IsInvisible = on == true
		pcall(function()
			local b = _G.SosySedseBridge
			if b and b.setSedse then b.setSedse("Invisible", on == true) end
		end)
		pcall(function() setSedse("Invisible", on == true) end)
		if type(_G.SetInvisible) == "function" then
			pcall(_G.SetInvisible, on == true)
		end
	end

	local function setRapidTp(on)
		_G.AutoKillState = _G.AutoKillState or {}
		_G.AutoKillState.tpActive = on == true
		pcall(function()
			local b = _G.SosySedseBridge
			if b and b.setSedse then
				b.setSedse("Rapid TP", on == true)
				b.setSedse("Rapid Teleport", on == true)
			end
		end)
		pcall(function()
			setSedse("Rapid TP", on == true)
			setSedse("Rapid Teleport", on == true)
		end)
	end

	local function hasActiveDomain()
		local folder = workspace:FindFirstChild("Domains")
		if not folder then return false end
		if folder:FindFirstChild("Domain") then return true end
		return #folder:GetChildren() > 0
	end

	local function ultReady()
		local ult = lp:GetAttribute("Ultimate")
		if typeof(ult) ~= "number" then ult = 0 end
		return ult >= 100
	end

	local function getMyChar()
		local folder = workspace:FindFirstChild("Characters")
		if folder then
			local c = folder:FindFirstChild(lp.Name)
			if c then return c end
		end
		return lp.Character
	end

	local function nearestEnemy()
		local me = getMyChar()
		local myHrp = me and me:FindFirstChild("HumanoidRootPart")
		if not myHrp then return nil end
		local folder = workspace:FindFirstChild("Characters")
		if not folder then return nil end
		local best, bestDist = nil, 1e9
		for _, m in ipairs(folder:GetChildren()) do
			if m ~= me and m:IsA("Model") then
				local hrp = m:FindFirstChild("HumanoidRootPart")
				local hum = m:FindFirstChildOfClass("Humanoid")
				if hrp and hum and hum.Health > 0 then
					local d = (hrp.Position - myHrp.Position).Magnitude
					if d < bestDist then
						bestDist = d
						best = hrp
					end
				end
			end
		end
		return best
	end

	local function chaseAndPunch()
		local me = getMyChar()
		local myHrp = me and me:FindFirstChild("HumanoidRootPart")
		if not myHrp then return end
		local target = nearestEnemy()
		if not target then return end
		pcall(function()
			myHrp.CFrame = target.CFrame * CFrame.new(0, 0, 3.2)
			myHrp.AssemblyLinearVelocity = Vector3.zero
		end)
		local now = realNow()
		if now - lastPunch >= 0.18 then
			lastPunch = now
			fireMelee()
		end
	end

	local function cast02Domain()
		if casting then return end
		if hasActiveDomain() then return end
		if not ultReady() then return end
		local now = realNow()
		if now - lastCast < 1.2 then return end
		lastCast = now
		casting = true
		_G.SpecialsState.Domain02Phase = "cast"
		task.spawn(function()
			tapKey(Enum.KeyCode.G, 0.05)
			task.wait(0.08)
			for _ = 1, 18 do
				if not (_G.SpecialsState and _G.SpecialsState.Domain02Farm) then break end
				if hasActiveDomain() then break end
				tapKey(Enum.KeyCode.R, 0.025)
				task.wait(0.03)
			end
			casting = false
		end)
	end

	local function onDomainOpened()
		if domainLatched then return end
		domainLatched = true
		_G.SpecialsState.Domain02Phase = "domain"
		setInvisible(true)
		setRapidTp(true)
	end

	function _G.SetDomain02Farm(v)
		ensure("SpecialsState", { Domain02Farm = false, Domain02Phase = "idle" })
		_G.SpecialsState.Domain02Farm = v == true
		if farmConn then
			pcall(function() farmConn:Disconnect() end)
			farmConn = nil
		end
		casting = false
		domainLatched = false
		if not _G.SpecialsState.Domain02Farm then
			_G.SpecialsState.Domain02Phase = "idle"
			return
		end
		_G.SpecialsState.Domain02Phase = "farm"
		setInvisible(true)
		farmConn = RunService.Heartbeat:Connect(function()
			if not (_G.SpecialsState and _G.SpecialsState.Domain02Farm) then return end
			if hasActiveDomain() then
				onDomainOpened()
				return
			end
			domainLatched = false
			if ultReady() then
				cast02Domain()
			else
				_G.SpecialsState.Domain02Phase = "farm"
				chaseAndPunch()
			end
		end)
	end

	task.defer(function()
		if _G.SpecialsState and _G.SpecialsState.Domain02Farm then
			_G.SetDomain02Farm(true)
		end
	end)
end
-- ===================== /0.2 Domain Farm =====================

-- ===================== Piano (SedseXD open-source MIDI engine) =====================
-- Source: https://raw.githubusercontent.com/SedseXD/piano/refs/heads/main/pianoscript.lua
do
	local vim = game:GetService("VirtualInputManager")
	local SONG_DIRS = {
		"SosyHub/Piano/Songs",
		"SosyHUB/Piano/Songs",
		"RobloxPiano/Songs",
		"midi",
		"Piano/Songs",
		"Songs",
		"workspace/SosyHub/Piano/Songs",
		"workspace/SosyHUB/Piano/Songs",
		"workspace/RobloxPiano/Songs",
		"workspace/midi",
	}

	local function fs()
		local env = (getgenv and getgenv()) or (getfenv and getfenv()) or _G
		return {
			listfiles = listfiles or env.listfiles,
			isfile = isfile or env.isfile,
			isfolder = isfolder or env.isfolder,
			makefolder = makefolder or env.makefolder,
			writefile = writefile or env.writefile,
			readfile = readfile or env.readfile,
		}
	end

	local function ensureSongFolders()
		local F = fs()
		pcall(function()
			if not F.makefolder then return end
			local function mk(path)
				if F.isfolder and F.isfolder(path) then return end
				F.makefolder(path)
			end
			mk("SosyHub"); mk("SosyHub/Piano"); mk("SosyHub/Piano/Songs")
			mk("SosyHUB"); mk("SosyHUB/Piano"); mk("SosyHUB/Piano/Songs")
			mk("RobloxPiano"); mk("RobloxPiano/Songs")
			pcall(function()
				if F.writefile then
					F.writefile(
						"SosyHub/Piano/Songs/PUT_MIDI_HERE.txt",
						"Put .mid / .midi files in this folder, then press Scan Songs in Piano tab.\n"
					)
				end
			end)
		end)
	end
	ensureSongFolders()

	_G.PianoState = _G.PianoState or {}
	local PS = _G.PianoState
	PS.Status = PS.Status or "Stopped"
	PS.Speed = tonumber(PS.Speed) or 1
	PS.Transpose = tonumber(PS.Transpose) or 0
	PS.Looping = PS.Looping == true
	PS.PianoSongsPath = PS.PianoSongsPath or "SosyHub/Piano/Songs"
	PS.FileMap = PS.FileMap or {}
	PS.Song = PS.Song or {}
	PS.Idx = PS.Idx or 1
	PS.LastTime = PS.LastTime or 0
	PS.Loading = false

	-- MIDI parser (SedseXD)
	local function parseMidi(data)
		local pos = 1
		local function rb()
			local b = string.byte(data, pos)
			pos = pos + 1
			return b
		end
		local function rs(len)
			local s = string.sub(data, pos, pos + len - 1)
			pos = pos + len
			return s
		end
		local function r16()
			return rb() * 256 + rb()
		end
		local function r32()
			return rb() * 16777216 + rb() * 65536 + rb() * 256 + rb()
		end
		local function rvlq()
			local v = 0
			while true do
				local b = rb()
				v = v * 128 + bit32.band(b, 0x7F)
				if bit32.band(b, 0x80) == 0 then break end
			end
			return v
		end

		if rs(4) ~= "MThd" then return {} end
		r32()
		local _fmt, ntrk, div = r16(), r16(), r16()
		if not div or div == 0 then return {} end
		local evs = {}

		-- Parsing runs on whichever thread selected the song. A few thousand events
		-- is enough to blow a frame budget, which is what froze the game on pick.
		local budget = 0
		local function breathe()
			budget = budget + 1
			if budget >= 3000 then
				budget = 0
				task.wait()
			end
		end

		for _ = 1, ntrk do
			if rs(4) ~= "MTrk" then break end
			local len = r32()
			local endp, tick, lst = pos + len, 0, 0
			while pos < endp do
				breathe()
				local d = rvlq()
				tick = tick + d
				local s = rb()
				if s < 0x80 then
					s = lst
					pos = pos - 1
				else
					lst = s
				end
				local et = bit32.rshift(s, 4)
				local ch = bit32.band(s, 0x0F)
				if s == 0xFF then
					local mt = rb()
					local ml = rvlq()
					local md = rs(ml)
					if mt == 0x51 and ml == 3 then
						local mpqn = string.byte(md, 1) * 65536 + string.byte(md, 2) * 256 + string.byte(md, 3)
						table.insert(evs, { tick = tick, type = "t", mpqn = mpqn })
					end
				elseif et == 0x9 then
					local n, v = rb(), rb()
					if v > 0 and ch ~= 9 then table.insert(evs, { tick = tick, type = "n", note = n }) end
				elseif et == 0x8 or et == 0xA or et == 0xB or et == 0xE then
					rb()
					rb()
				elseif et == 0xC or et == 0xD then
					rb()
				elseif s == 0xF0 or s == 0xF7 then
					-- SysEx: skip VLQ-length bytes and clear running status
					local sl = rvlq(); pos = pos + sl; lst = 0
				elseif s == 0xF2 then
					rb(); rb()
				elseif s == 0xF1 or s == 0xF3 then
					rb()
				end
			end
			pos = endp
		end

		table.sort(evs, function(a, b) return a.tick < b.tick end)
		local res, curt, curtm, mpqn = {}, 0, 0, 500000
		for _, e in ipairs(evs) do
			breathe()
			local dt = e.tick - curt
			if dt > 0 then
				curtm = curtm + (dt / div) * (mpqn / 1000000)
				curt = e.tick
			end
			if e.type == "t" then
				mpqn = e.mpqn
			elseif e.type == "n" then
				table.insert(res, { time = curtm, note = e.note })
			end
		end
		return res
	end

	-- Standard 61-key Virtual Piano layout, fully chromatic from MIDI 36 (C2) to 96 (C7).
	-- The previous table started at 60 and skipped notes (120 had no entry at all), so
	-- songs played about two octaves low and dropped keys mid-phrase.
	local PIANO_LAYOUT = {
		"1", "!", "2", "@", "3", "4", "$", "5", "%", "6", "^", "7",
		"8", "*", "9", "(", "0", "q", "Q", "w", "W", "e", "E", "r",
		"t", "T", "y", "Y", "u", "i", "I", "o", "O", "p", "P", "a",
		"s", "S", "d", "D", "f", "g", "G", "h", "H", "j", "J", "k",
		"l", "L", "z", "Z", "x", "c", "C", "v", "V", "b", "B", "n",
		"m",
	}
	local PIANO_LOW, PIANO_HIGH = 36, 96

	local nm = {}
	for i, k in ipairs(PIANO_LAYOUT) do
		nm[PIANO_LOW + i - 1] = k
	end

	-- Fold out-of-range notes by octaves rather than dropping them, otherwise bass
	-- and lead lines vanish on songs that sit outside the 61-key window.
	local function keyFor(note)
		local n = tonumber(note)
		if not n then return nil end
		while n < PIANO_LOW do n = n + 12 end
		while n > PIANO_HIGH do n = n - 12 end
		return nm[n]
	end

	local km = {
		["1"] = Enum.KeyCode.One, ["2"] = Enum.KeyCode.Two, ["3"] = Enum.KeyCode.Three,
		["4"] = Enum.KeyCode.Four, ["5"] = Enum.KeyCode.Five, ["6"] = Enum.KeyCode.Six,
		["7"] = Enum.KeyCode.Seven, ["8"] = Enum.KeyCode.Eight, ["9"] = Enum.KeyCode.Nine, ["0"] = Enum.KeyCode.Zero,
		["q"] = Enum.KeyCode.Q, ["w"] = Enum.KeyCode.W, ["e"] = Enum.KeyCode.E, ["r"] = Enum.KeyCode.R,
		["t"] = Enum.KeyCode.T, ["y"] = Enum.KeyCode.Y, ["u"] = Enum.KeyCode.U, ["i"] = Enum.KeyCode.I,
		["o"] = Enum.KeyCode.O, ["p"] = Enum.KeyCode.P, ["a"] = Enum.KeyCode.A, ["s"] = Enum.KeyCode.S,
		["d"] = Enum.KeyCode.D, ["f"] = Enum.KeyCode.F, ["g"] = Enum.KeyCode.G, ["h"] = Enum.KeyCode.H,
		["j"] = Enum.KeyCode.J, ["k"] = Enum.KeyCode.K, ["l"] = Enum.KeyCode.L, ["z"] = Enum.KeyCode.Z,
		["x"] = Enum.KeyCode.X, ["c"] = Enum.KeyCode.C, ["v"] = Enum.KeyCode.V, ["b"] = Enum.KeyCode.B,
		["n"] = Enum.KeyCode.N, ["m"] = Enum.KeyCode.M,
		["!"] = Enum.KeyCode.One, ["@"] = Enum.KeyCode.Two, ["$"] = Enum.KeyCode.Four,
		["%"] = Enum.KeyCode.Five, ["^"] = Enum.KeyCode.Six, ["*"] = Enum.KeyCode.Eight, ["("] = Enum.KeyCode.Nine,
		["Q"] = Enum.KeyCode.Q, ["W"] = Enum.KeyCode.W, ["E"] = Enum.KeyCode.E, ["T"] = Enum.KeyCode.T,
		["Y"] = Enum.KeyCode.Y, ["I"] = Enum.KeyCode.I, ["O"] = Enum.KeyCode.O, ["P"] = Enum.KeyCode.P,
		["S"] = Enum.KeyCode.S, ["D"] = Enum.KeyCode.D, ["G"] = Enum.KeyCode.G, ["H"] = Enum.KeyCode.H,
		["J"] = Enum.KeyCode.J, ["L"] = Enum.KeyCode.L, ["Z"] = Enum.KeyCode.Z, ["C"] = Enum.KeyCode.C,
		["V"] = Enum.KeyCode.V, ["B"] = Enum.KeyCode.B,
	}

	local function isShift(k)
		return k:match("^[A-Z]$") or k:match("^[!@$%%^*(]$")
	end

	-- Track held VIM keys so we never leave piano UI keys stuck "pressed"
	local heldKeys = {}
	local shiftHeld = false

	local function keyUp(code)
		if not code then return end
		if heldKeys[code] then
			heldKeys[code] = nil
			pcall(function() vim:SendKeyEvent(false, code, false, game) end)
		end
	end

	local function keyDown(code)
		if not code then return end
		heldKeys[code] = true
		pcall(function() vim:SendKeyEvent(true, code, false, game) end)
	end

	local function releaseAllKeys()
		for code in pairs(heldKeys) do
			pcall(function() vim:SendKeyEvent(false, code, false, game) end)
		end
		table.clear(heldKeys)
		if shiftHeld then
			pcall(function() vim:SendKeyEvent(false, Enum.KeyCode.LeftShift, false, game) end)
			shiftHeld = false
		end
		-- belt-and-suspenders: release shift even if flag drifted
		pcall(function() vim:SendKeyEvent(false, Enum.KeyCode.LeftShift, false, game) end)
	end

	-- Synchronous chord play (NO task.spawn) — overlapping spawns caused stuck keys + late/mixed audio
	local function playChord(ks)
		if type(ks) ~= "table" or #ks == 0 then return end
		-- dedupe
		local seen, uniq = {}, {}
		for _, k in ipairs(ks) do
			if type(k) == "string" and not seen[k] then
				seen[k] = true
				table.insert(uniq, k)
			end
		end
		local un, sh = {}, {}
		for _, k in ipairs(uniq) do
			local c = km[k]
			if c then
				if isShift(k) then table.insert(sh, c) else table.insert(un, c) end
			end
		end
		-- Always clear previous presses first (fixes "keys lit but sound late/wrong")
		releaseAllKeys()
		local HOLD = 0.02
		-- A note registers on its key-down edge, so shift can be raised straight after
		-- the naturals without re-colouring them. Running both groups inside a single
		-- hold keeps a mixed chord near 20ms instead of ~60ms — the old two-phase
		-- version blocked long enough that fast passages fell behind the scheduler and
		-- got re-anchored mid-phrase, which is what made playback sound scrambled.
		for _, c in ipairs(un) do keyDown(c) end
		if #sh > 0 then
			shiftHeld = true
			pcall(function() vim:SendKeyEvent(true, Enum.KeyCode.LeftShift, false, game) end)
			for _, c in ipairs(sh) do keyDown(c) end
		end
		task.wait(HOLD)
		for _, c in ipairs(un) do keyUp(c) end
		for _, c in ipairs(sh) do keyUp(c) end
		if shiftHeld then
			pcall(function() vim:SendKeyEvent(false, Enum.KeyCode.LeftShift, false, game) end)
			shiftHeld = false
		end
	end

	-- Executors disagree about path separators: some accept "a/b/c", some only
	-- "a\b\c", and several have an isfolder() that answers false for the style they
	-- did not expect. The old scanner gated every directory behind isfolder() and
	-- only ever descended one level, so on a picky executor it silently found
	-- nothing even with a full Songs folder. These helpers never let isfolder()
	-- veto an attempt and always try both separator styles.
	local function pathVariants(dir)
		dir = tostring(dir or "")
		if dir == "" then return {} end
		local fwd = dir:gsub("\\", "/")
		local back = dir:gsub("/", "\\")
		if fwd == back then return { fwd } end
		return { fwd, back }
	end

	local function tryListDir(F, dir)
		if not F.listfiles then return nil end
		for _, variant in ipairs(pathVariants(dir)) do
			local ok, files = pcall(F.listfiles, variant)
			if ok and type(files) == "table" then return files, variant end
		end
		return nil
	end

	local function loadSongFromPath(path)
		if type(path) ~= "string" or path == "" then return false, "no path" end
		local ok, data = pcall(readfile, path)
		if not ok or type(data) ~= "string" or #data < 8 then
			return false, "read failed"
		end
		local parsed = parseMidi(data)
		if type(parsed) ~= "table" or #parsed == 0 then
			return false, "empty midi"
		end
		PS.FilePath = path
		PS.Song = parsed
		PS.Idx = 1
		PS.LastTime = 0
		PS.Status = "Stopped"
		PS.LastError = nil
		return true, #parsed
	end

	function _G.pianoLoadSong(pathOrName)
		ensureSongFolders()
		local F = fs()
		local path = pathOrName or PS.FilePath
		if type(path) ~= "string" or path == "" then return false, "no path" end

		-- The dropdown hands us a bare base name, which always ends in .mid — the old
		-- "only look it up if it has no .mid" guard meant FileMap was never consulted
		-- and songs in subfolders could not be resolved at all.
		if not path:find("[/\\]") then
			local map = PS.FileMap or {}
			path = map[path] or path
		end

		local function readable(p)
			if not p or p == "" then return false end
			if F.isfile then
				for _, variant in ipairs(pathVariants(p)) do
					local ok, res = pcall(F.isfile, variant)
					if ok and res then return true, variant end
				end
				return false
			end
			return true
		end

		local ok, resolved = readable(path)
		if ok and resolved then path = resolved end

		if not ok then
			-- try the known folders, both separator styles
			local base = path:match("[^/\\]+$") or path
			for _, dir in ipairs(SONG_DIRS) do
				local cand = (dir:gsub("[/\\]$", "")) .. "/" .. base
				local hit, variant = readable(cand)
				if hit then
					path = variant or cand
					break
				end
			end
		end

		local okLoad, info = loadSongFromPath(path)
		return okLoad, info
	end

	function _G.pianoScanSongs()
		ensureSongFolders()
		local F = fs()
		local opts, map = {}, {}
		local dbg = { hasListfiles = F.listfiles ~= nil, listed = {}, entries = 0, added = 0 }

		local function isMidiName(path)
			local low = string.lower(tostring(path or ""))
			return low:sub(-4) == ".mid" or low:sub(-5) == ".midi"
		end
		local function addFile(f)
			local path = tostring(f)
			if not isMidiName(path) then return end
			local base = path:match("[^/\\]+$") or path
			if map[base] then return end
			map[base] = path
			dbg.added = dbg.added + 1
			table.insert(opts, base)
		end

		local seen = {}
		local function scanDir(dir, depth)
			if depth > 3 then return end
			local key = string.lower(tostring(dir or "")):gsub("\\", "/")
			if key == "" or seen[key] then return end
			seen[key] = true

			local files, usedPath = tryListDir(F, dir)
			if not files then return end
			table.insert(dbg.listed, usedPath .. " (" .. #files .. ")")

			for _, f in ipairs(files) do
				local path = tostring(f)
				dbg.entries = dbg.entries + 1
				if isMidiName(path) then
					addFile(path)
				elseif #opts < 500 then
					-- Might be a subfolder. Trust isfolder when it says yes, but when it
					-- says no (or is missing) fall back to "no file extension" and let
					-- the recursive listfiles call be the real test.
					local base = path:match("[^/\\]+$") or path
					local isDir = false
					if F.isfolder then
						for _, variant in ipairs(pathVariants(path)) do
							local ok, res = pcall(F.isfolder, variant)
							if ok and res then isDir = true break end
						end
					end
					if not isDir then isDir = not base:find("%.[%w]+$") end
					if isDir then scanDir(path, depth + 1) end
				end
			end
		end

		for _, dir in ipairs(SONG_DIRS) do
			scanDir(dir, 1)
		end

		-- Last resort: walk the executor workspace roots. The old version required a
		-- root entry to contain BOTH "piano" and "song", which no real folder name
		-- does ("RobloxPiano", "SosyHub", "midi" all fail that test).
		if #opts == 0 then
			for _, root in ipairs({ ".", "workspace", "" }) do
				local files = root ~= "" and select(1, tryListDir(F, root)) or nil
				if root == "" and F.listfiles then
					local ok, res = pcall(F.listfiles)
					if ok and type(res) == "table" then files = res end
				end
				if type(files) == "table" then
					for _, f in ipairs(files) do
						local path = tostring(f)
						if isMidiName(path) then
							addFile(path)
						else
							local low = string.lower(path:match("[^/\\]+$") or path)
							if low:find("piano") or low:find("song") or low:find("midi")
								or low:find("sosyhub") then
								scanDir(path, 1)
							end
						end
					end
				end
			end
		end

		table.sort(opts)
		PS.FileMap = map
		PS.PianoSongsPath = "SosyHub/Piano/Songs"
		PS.LastScanCount = #opts
		PS.ScanDebug = dbg
		return opts, map
	end

	-- Exposed so a failed scan can be diagnosed from the console without guessing.
	function _G.pianoScanReport()
		local d = PS.ScanDebug or {}
		local lines = {
			"listfiles available: " .. tostring(d.hasListfiles),
			"entries seen: " .. tostring(d.entries or 0),
			"midi added: " .. tostring(d.added or 0),
			"dirs listed:",
		}
		for _, v in ipairs(d.listed or {}) do
			table.insert(lines, "   " .. v)
		end
		local report = table.concat(lines, "\n")
		print("[SosyHUB][Piano] scan report\n" .. report)
		return report
	end

	-- The 61-key layout reuses w/a/s/d/q/e/z/x/c/v/b/n/m, and VirtualInputManager
	-- feeds those to the movement controller as well as the piano, so the character
	-- walks off while a song plays. Suspend player control for the duration.
	local pianoControlsSuspended = false
	local function setPianoControls(enabled)
		if enabled == not pianoControlsSuspended then return end
		pcall(function()
			local plr = game:GetService("Players").LocalPlayer
			local scripts = plr and plr:FindFirstChild("PlayerScripts")
			local mod = scripts and scripts:FindFirstChild("PlayerModule")
			if not mod then return end
			local controls = require(mod):GetControls()
			if not controls then return end
			if enabled then controls:Enable() else controls:Disable() end
		end)
		pianoControlsSuspended = not enabled
		if not enabled then
			pcall(function()
				local plr = game:GetService("Players").LocalPlayer
				local char = plr and plr.Character
				local hum = char and char:FindFirstChildOfClass("Humanoid")
				if hum then hum:Move(Vector3.new(0, 0, 0), false) end
			end)
		end
	end

	local function forceStopFlightForPiano()
		local fl = _G.MiscState and _G.MiscState.Flight
		if type(fl) == "table" then
			pcall(function() fl.Keybind = Enum.KeyCode.F13 end)
			if fl.IsFlying == true then
				local toggle = _G._SosyRawFlightToggle or _G.FlightToggleCallback
				if type(toggle) == "function" then
					pcall(toggle, false)
				else
					fl.IsFlying = false
				end
			end
		end
	end

	function _G.pianoStartPlayback()
		ensureSongFolders()
		-- A selection is still parsing on another thread; starting now would load the
		-- same file twice and race the state it writes. Parsing is only tens of ms, so
		-- wait it out instead of silently refusing to play — bailing here looked
		-- exactly like "I pressed Play and nothing happened".
		if PS.Loading then
			local deadline = tick() + 3
			repeat task.wait(0.05) until not PS.Loading or tick() > deadline
			if PS.Loading then
				PS.LastError = "still loading"
				warn("[SosyHUB][Piano] play ignored: song still parsing after 3s")
				return false
			end
		end
		forceStopFlightForPiano()
		releaseAllKeys()
		-- (re)load if needed
		if type(PS.Song) ~= "table" or #PS.Song == 0 then
			if type(PS.FilePath) ~= "string" or PS.FilePath == "" then
				PS.Status = "Stopped"
				PS.LastError = "no song selected"
				warn("[SosyHUB][Piano] play ignored: no song selected")
				return false
			end
			local ok, err = _G.pianoLoadSong(PS.FilePath)
			if not ok then
				PS.Status = "Stopped"
				PS.LastError = tostring(err)
				warn("[SosyHUB][Piano] load failed:", err, PS.FilePath)
				return false
			end
		end
		if PS.Status == "Paused" then
			forceStopFlightForPiano()
			-- re-anchor clock so resume stays in sync
			local song = PS.Song
			local idx = PS.Idx or 1
			local songT = (song and song[idx] and song[idx].time) or (PS.LastTime or 0)
			PS.PlayAnchorReal = tick()
			PS.PlayAnchorSong = songT
			setPianoControls(false)
			PS.Status = "Playing"
			return true
		end
		PS.Idx = 1
		PS.LastTime = 0
		PS.PlayAnchorReal = tick()
		PS.PlayAnchorSong = 0
		setPianoControls(false)
		PS.Status = "Playing"
		forceStopFlightForPiano()
		return true
	end

	function _G.pianoPausePlayback()
		if PS.Status == "Playing" then
			PS.Status = "Paused"
			releaseAllKeys()
			setPianoControls(true)
		end
	end

	function _G.pianoStopPlayback()
		PS.Status = "Stopped"
		PS.Idx = 1
		PS.LastTime = 0
		PS.PlayAnchorReal = nil
		PS.PlayAnchorSong = nil
		releaseAllKeys()
		setPianoControls(true)
	end

	-- Respawning rebuilds the control module with input switched back on, so a death
	-- mid-song would let the piano keys start walking the character again.
	if not _G._SosyPianoRespawnHook then
		_G._SosyPianoRespawnHook = true
		pcall(function()
			game:GetService("Players").LocalPlayer.CharacterAdded:Connect(function()
				task.wait(0.5)
				if _G.PianoState and _G.PianoState.Status == "Playing" then
					pianoControlsSuspended = false -- force the idempotent guard to re-apply
					setPianoControls(false)
				end
			end)
		end)
	end

	-- While Playing: keep fly off + Keybind dead (Sedse still listens to Keybind==X from VIM)
	if not _G._SosyPianoFlightWatch then
		_G._SosyPianoFlightWatch = true
		task.spawn(function()
			while true do
				if _G.PianoState and _G.PianoState.Status == "Playing" then
					forceStopFlightForPiano()
					task.wait(0.15)
				else
					task.wait(0.5)
				end
			end
		end)
	end

	-- End = stop piano
	if not _G._SosyPianoEndKeyHooked then
		_G._SosyPianoEndKeyHooked = true
		local UIS = game:GetService("UserInputService")
		UIS.InputBegan:Connect(function(input, gpe)
			if gpe then return end
			if input.KeyCode ~= Enum.KeyCode.End then return end
			if type(_G.pianoStopPlayback) == "function" then
				_G.pianoStopPlayback()
			end
		end)
	end

	-- Playback loop v3: absolute clock + sync chords (no overlapping key spam)
	_G._SosyPianoGen = (_G._SosyPianoGen or 0) + 1
	local myGen = _G._SosyPianoGen
	_G._SosyPianoLoopStarted_v3 = true
	_G._SosyPianoLoopStarted = true
	task.spawn(function()
		while _G._SosyPianoGen == myGen do
			local st = PS.Status
			local song = PS.Song
			if st == "Playing" and type(song) == "table" and #song > 0 then
				local idx = PS.Idx or 1
				if idx > #song then
					releaseAllKeys()
					if PS.Looping then
						PS.Idx, PS.LastTime = 1, 0
						PS.PlayAnchorReal = tick()
						PS.PlayAnchorSong = 0
					else
						PS.Status, PS.Idx, PS.LastTime = "Stopped", 1, 0
						PS.PlayAnchorReal, PS.PlayAnchorSong = nil, nil
						-- Song ran out on its own: hand movement back to the player.
						setPianoControls(true)
					end
				else
					local e = song[idx]
					local speed = math.max(0.05, tonumber(PS.Speed) or 1)
					if not PS.PlayAnchorReal then
						PS.PlayAnchorReal = tick()
						PS.PlayAnchorSong = e.time
					end
					-- Absolute song clock — chord hold time does not drift the timeline
					local targetReal = PS.PlayAnchorReal + (e.time - (PS.PlayAnchorSong or 0)) / speed
					local delay = targetReal - tick()
					if delay < 0 then
						-- Behind schedule: fire immediately but keep the original anchor so
						-- the timeline stays absolute and the tempo does not creep. Only a
						-- genuine stall (alt-tab, hitch) is worth re-basing the clock for;
						-- the old 150ms threshold re-anchored constantly during fast runs
						-- and slowly dragged the whole song out of time.
						if delay < -1 then
							PS.PlayAnchorReal = tick()
							PS.PlayAnchorSong = e.time
						end
						delay = 0
					end
					if delay > 0.001 then
						task.wait(math.min(delay, 0.5))
					end
					if PS.Status ~= "Playing" or _G._SosyPianoGen ~= myGen then
						releaseAllKeys()
					else
						-- tight chord window; avoid grabbing half the next bar
						local window = math.max(0.008, 0.015 / speed)
						local ck, j = {}, idx
						local transpose = tonumber(PS.Transpose) or 0
						while j <= #song and (song[j].time - e.time) <= window do
							local k = keyFor(song[j].note + transpose)
							if k then table.insert(ck, k) end
							j = j + 1
						end
						if #ck > 0 then
							playChord(ck) -- waits until keys released
						end
						PS.LastTime, PS.Idx = e.time, j
					end
				end
			else
				task.wait(0.1)
			end
			task.wait()
		end
		releaseAllKeys()
		-- Superseded by a newer generation (script re-run): never leave controls off.
		setPianoControls(true)
	end)
end
-- ===================== /Piano =====================

-- ===================== Auto Perfect Swap (Todo) =====================
do
	local UIS     = game:GetService("UserInputService")
	local VIM     = game:GetService("VirtualInputManager")
	local RS      = game:GetService("RunService")
	local Players = game:GetService("Players")
	local lp      = Players.LocalPlayer
	ensure("TodoState", { AutoPerfectClap = false })

	local RSt = game:GetService("ReplicatedStorage")
	local uisConn, fxConn = nil, nil

	-- R only arms; the follow-up is released by the server telling us a swap actually
	-- happened. The old version guessed from HumanoidRootPart movement and, failing
	-- that, punched anyway after 0.5s — so every R press threw a fist even when no
	-- swap occurred. TodoService.RE.Effects "Swap" carries
	-- (swapper, target, myNewCFrame, targetNewCFrame, flag), verified live, which is
	-- both a definite confirmation and the identity of the exact target.
	local armedUntil = 0
	local ARM_WINDOW = 1.5

	local function myChar()
		local chars = workspace:FindFirstChild("Characters")
		return (chars and chars:FindFirstChild(lp.Name)) or lp.Character
	end

	-- ToolController.GetTarget picks whichever character projects closest to your
	-- mouse point on screen, so after a swap — when you and the target have traded
	-- places — the camera is what decides who the follow-up lands on. Turning the body
	-- alone would not change the pick.
	-- Same guard as Auto Turn: steering the camera during a hold-to-aim skill throws
	-- the aim off, and writing HumanoidRootPart while the server applies knockback
	-- makes the two fight and flings you across the map.
	local function safeToSteer(char)
		local info = char and char:FindFirstChild("Info")
		if info and (info:FindFirstChild("InSkill") or info:FindFirstChild("Stun")) then return false end
		if char and (tonumber(char:GetAttribute("Ragdoll")) or 0) > 0 then return false end
		return true
	end

	local function faceTarget(target)
		local char = myChar()
		local hrp = char and char:FindFirstChild("HumanoidRootPart")
		local th  = target and (target:FindFirstChild("HumanoidRootPart") or target.PrimaryPart)
		if not (hrp and th) then return false end
		if not safeToSteer(char) then return false end
		pcall(function()
			local cam = workspace.CurrentCamera
			if cam then cam.CFrame = CFrame.new(cam.CFrame.Position, th.Position) end
			local p = hrp.Position
			local flat = Vector3.new(th.Position.X, p.Y, th.Position.Z)
			if (flat - p).Magnitude > 0.1 then hrp.CFrame = CFrame.new(p, flat) end
		end)
		return true
	end

	local function perfectFollowUp(target)
		local look = tonumber(_G.TodoState and _G.TodoState.PerfectClapLook) or 0.15
		look = math.clamp(look, 0, 0.4)

		if not faceTarget(target) then return end
		-- Hold the aim briefly: one snap gets overwritten by the camera update on the
		-- very next frame. Capped well under a quarter second so it reads as a flick
		-- and never takes the camera away from you.
		local until_ = tick() + look
		local hold
		hold = RS.RenderStepped:Connect(function()
			if tick() > until_ or not target.Parent then
				hold:Disconnect()
				return
			end
			faceTarget(target)
		end)

		task.wait(math.clamp(tonumber(_G.TodoState and _G.TodoState.PerfectClapDelay) or 0.05, 0, 0.3))

		-- Fire the melee with the target named explicitly rather than sending a mouse
		-- click and hoping the screen-space pick lands on the right body. This is the
		-- same call ToolController makes, minus the guesswork.
		local char = myChar()
		if not (char and target and target.Parent) then return end
		pcall(function()
			local svc = RSt.Knit.Knit.Services:FindFirstChild(tostring(char:GetAttribute("Moveset")) .. "Service")
			if svc and svc:FindFirstChild("RE") and svc.RE:FindFirstChild("Activated") then
				svc.RE.Activated:FireServer(nil, target)
			end
		end)
	end

	function _G.SetAutoPerfectClap(on)
		ensure("TodoState", { AutoPerfectClap = false, PerfectClapLook = 0.15, PerfectClapDelay = 0.05 })
		_G.TodoState.AutoPerfectClap = on == true
		if uisConn then pcall(function() uisConn:Disconnect() end); uisConn = nil end
		if fxConn then pcall(function() fxConn:Disconnect() end); fxConn = nil end
		armedUntil = 0
		if not _G.TodoState.AutoPerfectClap then return end

		-- Read the game's own Special binding rather than assuming R. It is an
		-- InputContext under ReplicatedStorage.Keybind.Combat and the player can
		-- rebind it — measured live as E on this account while this code still
		-- hardcoded R, which armed nothing.
		local function specialKeys()
			local keys = {}
			pcall(function()
				local sp = game:GetService("ReplicatedStorage").Keybind.Combat:FindFirstChild("Special")
				if not sp then return end
				for _, ib in ipairs(sp:GetChildren()) do
					if ib.KeyCode and ib.KeyCode ~= Enum.KeyCode.Unknown then
						keys[ib.KeyCode] = true
					end
				end
			end)
			if not next(keys) then keys[Enum.KeyCode.R] = true end
			return keys
		end

		local watched = specialKeys()
		-- Re-read on rebind so the arm key follows the setting without a hub reload.
		pcall(function()
			local sp = game:GetService("ReplicatedStorage").Keybind.Combat:FindFirstChild("Special")
			if sp then
				sp.DescendantAdded:Connect(function() watched = specialKeys() end)
				sp.DescendantRemoving:Connect(function()
					task.defer(function() watched = specialKeys() end)
				end)
			end
		end)

		-- Stage tracing, off unless _G.PerfectClapDebug() turns it on. Every fix so far
		-- has been reasoned rather than observed, so this reports which link breaks
		-- instead of guessing at the next one.
		local function dbg(msg)
			if _G.TodoState and _G.TodoState.PerfectClapDebug then
				warn("[PerfectClap] " .. msg)
			end
		end

		do
			local names = {}
			for k in pairs(watched) do names[#names + 1] = tostring(k):gsub("Enum.KeyCode.", "") end
			dbg("kuruldu, dinlenen tuslar: " .. table.concat(names, ", "))
		end

		uisConn = UIS.InputBegan:Connect(function(input)
			-- Deliberately not gated on gameProcessedEvent. The Special key is bound
			-- through an InputContext, so the game consumes it and UserInputService
			-- reports it as already processed — which is precisely the press we need
			-- to catch, and the old `if processed then return end` threw it away, so
			-- the arm never happened and the follow-up never fired.
			--
			-- Arming on its own is inert: the follow-up is released by the server's
			-- Swap effect, so a press that produces no swap does nothing at all. The
			-- only case worth excluding is typing the key into a chat or search box.
			if UIS:GetFocusedTextBox() then return end
			if watched[input.KeyCode] then
				armedUntil = tick() + ARM_WINDOW
				dbg("KOL KURULDU (" .. tostring(input.KeyCode):gsub("Enum.KeyCode.", "") .. ")")
			end
		end)

		pcall(function()
			fxConn = RSt.Knit.Knit.Services.TodoService.RE.Effects.OnClientEvent:Connect(function(name, swapper, target)
				if name ~= "Swap" then return end
				dbg("Swap geldi")
				-- Effects is broadcast, so other people's swaps arrive here too.
				if swapper ~= myChar() then
					dbg("  ELENDI: swap benim degil (" .. tostring(swapper) .. ")")
					return
				end
				if tick() > armedUntil then
					dbg("  ELENDI: kol kurulu degil — tus basimi yakalanmadi")
					return
				end
				armedUntil = 0 -- one follow-up per press
				if typeof(target) ~= "Instance" then
					dbg("  ELENDI: hedef instance degil")
					return
				end
				dbg("  M1 gonderiliyor -> " .. tostring(target))
				task.spawn(perfectFollowUp, target)
			end)
		end)
	end

	-- Turn tracing on, then press the clap key once. The console then names the exact
	-- link that fails instead of leaving us to guess at the next patch.
	function _G.PerfectClapDebug(on)
		ensure("TodoState", { AutoPerfectClap = false })
		_G.TodoState.PerfectClapDebug = (on ~= false)
		local keys = "?"
		pcall(function()
			local sp = game:GetService("ReplicatedStorage").Keybind.Combat:FindFirstChild("Special")
			local t = {}
			if sp then
				for _, ib in ipairs(sp:GetChildren()) do
					local k = tostring(ib.KeyCode):gsub("Enum.KeyCode.", "")
					if k ~= "Unknown" then t[#t + 1] = k end
				end
			end
			keys = table.concat(t, ", ")
		end)
		warn("[PerfectClap] iz surme = " .. tostring(_G.TodoState.PerfectClapDebug))
		warn("[PerfectClap] toggle    = " .. tostring(_G.TodoState.AutoPerfectClap))
		warn("[PerfectClap] oyun Special tusu = " .. keys)
		warn("[PerfectClap] simdi clap tusuna bir kez bas")
		return true
	end
end
-- ===================== /Auto Perfect Swap (Todo) =====================

-- ===================== Todo Fast Swap (client-side prediction) =====================
-- Measured in-game: firing TodoService.RE.RightActivated and waiting for the server
-- to send the swap back through TodoService.RE.Effects takes ~576-767 ms against a
-- living player, ~362 ms against a Throwable object. The whole delay is server-side:
-- ToolController fires the remote the instant you press the key (no client windup),
-- and TodoController's Swap FX — the thing that actually moves your HumanoidRootPart
-- — only runs when the server calls it back. No client flag can beat that round trip;
-- _G.Shift (shift-lock / lock-on) only decides whether the FX snaps your own HRP and
-- camera or lets replication do it, and measured identically (767/601 vs 576 ms).
-- The one thing that IS available client-side is prediction: a swap always ends with
-- you standing where the target stood, so we can go there ourselves the moment the
-- remote leaves, and let the server's swap converge onto the same spot ~0.5s later.
-- Verified live: arrival at 0 ms instead of ~590 ms, 1 stud from the target, no
-- rubber-band (0 studs off the target spot once the server swap landed).
do
	local Players = game:GetService("Players")
	local RS      = game:GetService("ReplicatedStorage")
	local lp      = Players.LocalPlayer
	ensure("TodoState", { FastSwap = false, FastSwapOffset = 1.5 })

	local lastPredictAt = 0

	local function myChar()
		local chars = workspace:FindFirstChild("Characters")
		return (chars and chars:FindFirstChild(lp.Name)) or lp.Character
	end

	local function targetCFrame(target)
		if typeof(target) ~= "Instance" then return nil end
		if target:IsA("BasePart") then return target.CFrame end
		if not target:IsA("Model") then return nil end
		local hrp = target:FindFirstChild("HumanoidRootPart") or target.PrimaryPart
		return hrp and hrp.CFrame or nil
	end

	-- Measured against the live server (TodoService.RE.Effects timestamps):
	--   player target : "Swap"  lands at 582-583 ms, dead consistent
	--   object target : "BlueGrab2" at 132 ms then "Swap2" at 331 ms
	-- 132 ms of that is round-trip latency, so the server's own wind-up is ~450 ms for
	-- a player and ~200 ms for an object. That gap is server-side and the only thing
	-- that selects between the two paths is the *instance* you pass: a Model under
	-- workspace.Characters vs one under Map.Destructible.Throwable / tagged
	-- CanBoogieWoogie. Both of those are server-replicated facts — a client cannot move
	-- a player into the Throwable folder, cannot add a replicating CollectionService
	-- tag, and cannot write an attribute upstream. So the 450 ms cannot be shortened.
	-- What we *can* do is stop waiting for it: you own your own HumanoidRootPart, so
	-- writing its CFrame replicates, and the server's Swap lands on the same spot later.

	local holdConn, holdUntil, holdCF = nil, 0, nil
	-- Measured by firing 8 swaps 400 ms apart: only 4 came back. Confirmed gaps were
	-- 725 / 804 / 934 ms, so the server honours roughly one swap per 750-950 ms and
	-- silently drops anything sent while it is still winding up. A fixed debounce
	-- cannot see that, so we track the request instead: predict only when nothing is
	-- in flight, because a prediction the server never honours is a teleport with no
	-- authority behind it and replication yanks you straight back.
	local swapInFlight = false
	local inFlightUntil = 0

	local function stopHold()
		if holdConn then pcall(function() holdConn:Disconnect() end) end
		holdConn, holdCF = nil, nil
	end

	local function clearInFlight()
		swapInFlight = false
		stopHold()
	end

	-- Exposed so Todo Bring / other features can reuse the same prediction.
	function _G.TodoPredictSwap(target)
		local st = _G.TodoState
		if not (st and st.FastSwap) then return false end

		local char = myChar()
		if not char then return false end
		if tostring(char:GetAttribute("Moveset") or "") ~= "Todo" then return false end

		local cf = targetCFrame(target)
		if not cf then return false end

		local hrp = char:FindFirstChild("HumanoidRootPart") or char.PrimaryPart
		if not hrp then return false end

		-- One prediction per honoured swap. If the previous request has not come back
		-- yet, this press is one the server is going to drop, so predicting it would
		-- move you with nothing to back it up.
		local now = tick()
		if swapInFlight and now < inFlightUntil then return false end
		if now - lastPredictAt < 0.35 then return false end
		lastPredictAt = now
		swapInFlight = true
		-- Fallback release: if the server never answers (out of range, stunned, dead
		-- target) no Swap effect arrives, so time the flag out rather than jamming.
		inFlightUntil = now + 1.1

		local offset = tonumber(st.FastSwapOffset) or 1.5
		local dest = cf * CFrame.new(0, 0, offset)

		pcall(function()
			-- Carry the camera across by the same relative transform the game's own
			-- Swap handler uses. Without this the body teleports and the camera is left
			-- behind until replication catches up, which is what makes an otherwise
			-- instant swap still feel late.
			local cam = workspace.CurrentCamera
			local rel = cam and hrp.CFrame:ToObjectSpace(cam.CFrame) or nil

			hrp.AssemblyLinearVelocity = Vector3.zero
			hrp.CFrame = dest
			if cam and rel then cam.CFrame = dest:ToWorldSpace(rel) end
		end)

		-- Hold the spot until the server's own Swap arrives (~580 ms). Until then the
		-- server still believes you are at the old position and its replication stream
		-- will drag you back; re-asserting each frame turns that rubber-band into a
		-- clean teleport.
		stopHold()
		holdCF, holdUntil = dest, tick() + 0.75
		holdConn = game:GetService("RunService").Heartbeat:Connect(function()
			if tick() > holdUntil or not holdCF then stopHold() return end
			local c = myChar()
			local h = c and (c:FindFirstChild("HumanoidRootPart") or c.PrimaryPart)
			if not h then stopHold() return end
			-- Only correct real drift, so normal movement input still works.
			if (h.Position - holdCF.Position).Magnitude > 6 then
				pcall(function() h.CFrame = holdCF end)
			end
		end)
		return true
	end

	-- The server's Swap/Swap2 effect is the authoritative "you are there now" signal;
	-- once it lands there is nothing left to hold against.
	task.spawn(function()
		local ok, re = pcall(function()
			return RS.Knit.Knit.Services.TodoService.RE.Effects
		end)
		if not ok or not re then return end
		re.OnClientEvent:Connect(function(name)
			if name == "Swap" or name == "Swap2" then clearInFlight() end
		end)
	end)

	function _G.SetTodoFastSwap(on)
		ensure("TodoState", { FastSwap = false, FastSwapOffset = 1.5 })
		_G.TodoState.FastSwap = on == true
		if not _G.TodoState.FastSwap then return end

		-- Hook the remote itself rather than ToolController: this way we only predict
		-- when a swap request actually goes out, whatever fired it (your keybind, the
		-- mobile button, or Todo Bring).
		if _G._SosyTodoSwapHooked then return end
		if typeof(hookmetamethod) ~= "function" or typeof(getnamecallmethod) ~= "function" then
			warn("[SosyHUB][Todo] Fast Swap needs hookmetamethod/getnamecallmethod; executor lacks them")
			return
		end
		_G._SosyTodoSwapHooked = true

		local cachedRE = nil
		local function swapRE()
			if cachedRE and cachedRE.Parent then return cachedRE end
			pcall(function()
				cachedRE = RS.Knit.Knit.Services.TodoService.RE.RightActivated
			end)
			return cachedRE
		end

		local old
		local function handler(self, ...)
			local re = swapRE()
			if re and self == re and string.lower(getnamecallmethod()) == "fireserver" then
				local target = ...
				-- Defer so we never re-enter the namecall hook from inside it.
				task.defer(function()
					pcall(_G.TodoPredictSwap, target)
				end)
			end
			return old(self, ...)
		end
		if typeof(newcclosure) == "function" then handler = newcclosure(handler) end
		old = hookmetamethod(game, "__namecall", handler)
	end

	-- ── Clap animation speed (COSMETIC ONLY) ─────────────────────────────────
	-- Measured: the clap track is id 131358603583212, Length 0.800s played at Speed
	-- 1.25 (= 0.64s), which lines up with the ~590 ms swap delay. Speeding the local
	-- track up does NOT bring the swap forward — trials at Speed 6 landed at 576 and
	-- 629 ms against 600 ms at the default, i.e. inside the noise. The server runs its
	-- own timer and ignores your local animation. This exists only so you are not left
	-- standing in a half-second clap after Fast Swap has already moved you.
	local CLAP_ANIM_ID = "131358603583212"
	local clapAnimConns = {}

	local function dropClapConns()
		for i = #clapAnimConns, 1, -1 do
			pcall(function() clapAnimConns[i]:Disconnect() end)
			clapAnimConns[i] = nil
		end
	end

	local function hookClapAnim(char)
		if not char then return end
		local hum = char:FindFirstChildOfClass("Humanoid")
		local animator = hum and hum:FindFirstChildOfClass("Animator")
		if not animator then return end
		clapAnimConns[#clapAnimConns + 1] = animator.AnimationPlayed:Connect(function(track)
			local st = _G.TodoState
			if not (st and st.FastSwapAnim) then return end
			local id = tostring(track.Animation and track.Animation.AnimationId or "")
			if not id:find(CLAP_ANIM_ID, 1, true) then return end
			local mult = tonumber(st.FastSwapAnimSpeed) or 3
			pcall(function() track:AdjustSpeed(1.25 * math.max(1, mult)) end)
		end)
	end

	function _G.SetTodoFastSwapAnim(on)
		ensure("TodoState", { FastSwapAnim = false, FastSwapAnimSpeed = 3 })
		_G.TodoState.FastSwapAnim = on == true
		dropClapConns()
		if not _G.TodoState.FastSwapAnim then return end
		hookClapAnim(myChar())
		if not _G._SosyTodoClapRespawnHook then
			_G._SosyTodoClapRespawnHook = true
			lp.CharacterAdded:Connect(function(char)
				if not (_G.TodoState and _G.TodoState.FastSwapAnim) then return end
				char:WaitForChild("Humanoid", 10)
				task.wait(0.3)
				dropClapConns()
				hookClapAnim(myChar())
			end)
		end
	end

	task.defer(function()
		if _G.TodoState and _G.TodoState.FastSwap then
			pcall(_G.SetTodoFastSwap, true)
		end
		if _G.TodoState and _G.TodoState.FastSwapAnim then
			pcall(_G.SetTodoFastSwapAnim, true)
		end
	end)
end
-- ===================== /Todo Fast Swap =====================

-- ===================== Anti Domain =====================
do
	local RS      = game:GetService("RunService")
	local Players = game:GetService("Players")
	local lp      = Players.LocalPlayer
	ensure("MiscState", { AntiDomain = false })

	local _antiDomainConn = nil
	local _domainEscapeConn = nil

	local function getChar()
		local chars = workspace:FindFirstChild("Characters")
		if chars then
			local c = chars:FindFirstChild(lp.Name)
			if c then return c end
		end
		return lp.Character
	end

	local function getNearestSpawn()
		local spawns = workspace:FindFirstChild("Spawns")
		if not spawns then return nil end
		local myChar = getChar()
		local myHrp = myChar and myChar:FindFirstChild("HumanoidRootPart")
		if not myHrp then return nil end
		local best, bestDist = nil, 1e9
		for _, s in ipairs(spawns:GetDescendants()) do
			if s:IsA("SpawnLocation") or (s:IsA("BasePart") and s.Name:lower():find("spawn")) then
				local d = (s.Position - myHrp.Position).Magnitude
				if d < bestDist then bestDist = d; best = s end
			end
		end
		return best
	end

	local function clearDomainDebuffs()
		local char = getChar()
		if not char then return end
		pcall(function()
			local info = char:FindFirstChild("Info")
			if not info then return end
			for _, child in ipairs(info:GetChildren()) do
				local n = string.lower(child.Name)
				if n:find("domain") or n:find("clash") or n:find("domainclash") then
					pcall(function() child:Destroy() end)
				end
			end
		end)
		pcall(function()
			if char:GetAttribute("DomainBind") ~= nil then
				char:SetAttribute("DomainBind", nil)
			end
		end)
	end

	local function escapeActiveDomain()
		local domainsFolder = workspace:FindFirstChild("Domains")
		if not domainsFolder or #domainsFolder:GetChildren() == 0 then return end
		local char = getChar()
		local hrp = char and char:FindFirstChild("HumanoidRootPart")
		if not hrp then return end
		local spawn = getNearestSpawn()
		if spawn then
			pcall(function()
				hrp.AssemblyLinearVelocity = Vector3.zero
				hrp.CFrame = CFrame.new(spawn.Position + Vector3.new(0, 5, 0))
			end)
		end
		clearDomainDebuffs()
	end

	function _G.SetAntiDomain(on)
		ensure("MiscState", { AntiDomain = false })
		_G.MiscState.AntiDomain = on == true
		if _antiDomainConn then pcall(function() _antiDomainConn:Disconnect() end); _antiDomainConn = nil end
		if _domainEscapeConn then pcall(function() _domainEscapeConn:Disconnect() end); _domainEscapeConn = nil end
		if not _G.MiscState.AntiDomain then return end
		local debuffAcc = 0
		_antiDomainConn = RS.Heartbeat:Connect(function(dt)
			if not (_G.MiscState and _G.MiscState.AntiDomain) then return end
			debuffAcc = debuffAcc + dt
			if debuffAcc < 0.25 then return end
			debuffAcc = 0
			clearDomainDebuffs()
		end)
		local domainsFolder = workspace:FindFirstChild("Domains")
		if domainsFolder then
			_domainEscapeConn = domainsFolder.ChildAdded:Connect(function()
				if not (_G.MiscState and _G.MiscState.AntiDomain) then return end
				task.wait(0.05)
				escapeActiveDomain()
			end)
		end
	end
end
-- ===================== /Anti Domain =====================

-- ===================== Auto Skills (generic) =====================
-- Auto Ratio and Auto Final Judgement are the same shape — wait for a moveset move to
-- come off cooldown, check a target is in range, fire it — so they share one engine
-- and differ only by which Service they look for. Adding another auto-move later is a
-- single AUTO_SKILLS entry.
--
-- Fired through ToolController:UseTool rather than by calling <Move>Service.RE
-- .Activated directly: ToolController assembles a different argument shape per move
-- (aerial flag, target, mouse target, move vector, or a table of all four) chosen from
-- an internal per-move table, so a hand-built remote call would be rejected or behave
-- wrongly. Verified live on Ratio Breaker — UseTool returned true, LastUse advanced,
-- and the service emitted Start / ArmCharge / Whoosh2.
--
-- VirtualInputManager is deliberately not used to press the skill key: this game binds
-- keys through InputContext objects and synthetic VIM presses do not drive them
-- (measured — neither R nor E produced anything through VIM).
do
	local RunService = game:GetService("RunService")
	local Players    = game:GetService("Players")
	local RSt        = game:GetService("ReplicatedStorage")
	local lp         = Players.LocalPlayer

	ensure("CharState", { AutoSkillRange = 60 })

	local AUTO_SKILLS = {
		{ flag = "AutoRatio",          service = "RatioBreakerService"   },
		{ flag = "AutoFinalJudgement", service = "FinalJudgementService" },
	}

	local conns, lastFire = {}, {}

	local function myChar()
		local chars = workspace:FindFirstChild("Characters")
		return (chars and chars:FindFirstChild(lp.Name)) or lp.Character
	end

	-- ReadyAt is stamped against the server clock, so compare on the same clock.
	-- Referenced with a dot: `workspace:GetServerTimeNow and` is a syntax error,
	-- because method-call syntax must be followed by the call itself.
	local function serverNow()
		if type(workspace.GetServerTimeNow) == "function" then
			local ok, t = pcall(function() return workspace:GetServerTimeNow() end)
			if ok and t then return t end
		end
		return os.time()
	end

	-- Moveset children carry Key / Service / LastUse / ReadyAt. Service is the stable
	-- identifier; display names differ per stance, and a move is only present in the
	-- stance that owns it — so a missing entry means "not available right now", not a
	-- failure.
	local function findEntry(char, serviceName)
		local ms = char:FindFirstChild("Moveset")
		if not ms then return nil end
		for _, e in ipairs(ms:GetChildren()) do
			if tostring(e:GetAttribute("Service") or "") == serviceName then return e end
		end
		return nil
	end

	local function offCooldown(entry)
		local readyAt = tonumber(entry:GetAttribute("ReadyAt"))
		if not readyAt then return true end
		return serverNow() >= readyAt
	end

	local function actionable(char)
		local info = char:FindFirstChild("Info")
		if info and (info:FindFirstChild("Stun") or info:FindFirstChild("InSkill")) then return false end
		local hum = char:FindFirstChildOfClass("Humanoid")
		if hum and hum.Health <= 0 then return false end
		return true
	end

	local function targetInRange(char)
		local hrp = char:FindFirstChild("HumanoidRootPart")
		local chars = workspace:FindFirstChild("Characters")
		if not (hrp and chars) then return false end
		local range = tonumber(_G.CharState and _G.CharState.AutoSkillRange) or 60
		for _, c in ipairs(chars:GetChildren()) do
			local h = c ~= char and c:FindFirstChild("HumanoidRootPart")
			local hum = h and c:FindFirstChildOfClass("Humanoid")
			if h and hum and hum.Health > 0 and (h.Position - hrp.Position).Magnitude <= range then
				return true
			end
		end
		return false
	end

	local function fire(entry)
		return pcall(function()
			local tc = require(RSt.Knit.Knit).GetController("ToolController")
			if tc and type(tc.UseTool) == "function" then tc:UseTool(entry, "Activated") end
		end)
	end

	local function setSkill(def, on)
		_G[def.flag] = on == true
		if conns[def.flag] then
			pcall(function() conns[def.flag]:Disconnect() end)
			conns[def.flag] = nil
		end
		if not _G[def.flag] then return end

		conns[def.flag] = RunService.Heartbeat:Connect(function()
			if not _G[def.flag] then return end
			local now = tick()
			if now - (lastFire[def.flag] or 0) < 0.35 then return end

			local char = myChar()
			if not char or not actionable(char) then return end

			local entry = findEntry(char, def.service)
			if not entry or not offCooldown(entry) then return end
			if not targetInRange(char) then return end

			lastFire[def.flag] = now
			fire(entry)
		end)
	end

	for _, def in ipairs(AUTO_SKILLS) do
		_G["Set" .. def.flag] = function(on) setSkill(def, on) end
	end

	-- Tells a silent no-op apart from a broken feature: most often the move simply is
	-- not in the current stance.
	function _G.AutoSkillReport()
		local char = myChar()
		warn("[AutoSkill] moveset = " .. tostring(char and char:GetAttribute("Moveset")))
		for _, def in ipairs(AUTO_SKILLS) do
			local entry = char and findEntry(char, def.service)
			warn(string.format("[AutoSkill] %s: acik=%s | moveset'te=%s%s",
				def.flag, tostring(_G[def.flag] == true), tostring(entry ~= nil),
				entry and (" | key=" .. tostring(entry:GetAttribute("Key"))
					.. " | hazir=" .. tostring(offCooldown(entry))) or ""))
		end
		warn("[AutoSkill] menzil = " .. tostring(_G.CharState and _G.CharState.AutoSkillRange))
		return true
	end
end
-- ===================== /Auto Skills (generic) =====================

-- ===================== No Dash Cooldowns =====================
-- The dash gate lives on the client, not the server. MovementController keeps the
-- last-dash timestamp in an upvalue and the game's own ResetDash handler clears it:
--
--     u10.ResetDash:Connect(function() u2 = tick() - 2 end)
--
-- ResetDash is a server -> client remote, so rather than reaching into the upvalue we
-- invoke that same handler locally through getconnections. It is the game's own reset
-- path, just triggered from here — verified live: one connection is present on
-- OnClientEvent and firing it returned cleanly.
--
-- Nothing is sent to the server, so this only removes the client-side wait; any
-- server-side rule still applies.
do
	local RunService = game:GetService("RunService")
	local RSt        = game:GetService("ReplicatedStorage")
	ensure("CharState", { NoDashCD = false, DashResetRate = 0.1 })

	local conn, lastReset = nil, 0
	local cachedConns = nil

	local function resetConnections()
		if cachedConns then return cachedConns end
		if type(getconnections) ~= "function" then return nil end
		local ok, list = pcall(function()
			local mv = RSt.Knit.Knit.Services:FindFirstChild("MovementService")
			local rd = mv and mv:FindFirstChild("RE") and mv.RE:FindFirstChild("ResetDash")
			if not rd then return nil end
			return getconnections(rd.OnClientEvent)
		end)
		if ok and type(list) == "table" and #list > 0 then cachedConns = list end
		return cachedConns
	end

	function _G.SetNoDashCD(on)
		ensure("CharState", { NoDashCD = false, DashResetRate = 0.1 })
		_G.CharState.NoDashCD = on == true
		if conn then pcall(function() conn:Disconnect() end); conn = nil end
		if not _G.CharState.NoDashCD then return end

		if type(getconnections) ~= "function" then
			warn("[SosyHUB] No Dash CD: bu executor getconnections desteklemiyor")
			return
		end

		conn = RunService.Heartbeat:Connect(function()
			if not (_G.CharState and _G.CharState.NoDashCD) then return end
			local now = tick()
			-- Ticking rather than firing every frame: the handler only rewinds a
			-- timestamp, so a few times a second is enough and keeps the cost flat.
			local rate = math.clamp(tonumber(_G.CharState.DashResetRate) or 0.1, 0.02, 1)
			if now - lastReset < rate then return end
			lastReset = now

			local list = resetConnections()
			if not list then return end
			for _, c in ipairs(list) do
				pcall(function() c:Fire() end)
			end
		end)
	end
end
-- ===================== /No Dash Cooldowns =====================

-- ===================== Auto Turn 180 on Swap =====================
-- After a Boogie Woogie swap you and the target have traded places, so you land still
-- facing your old direction with the target behind you. This turns you to them the
-- moment the server confirms the swap.
--
-- Driven off TodoService.RE.Effects "Swap", whose arguments were confirmed live as
-- (swapper, target, myNewCFrame, targetNewCFrame, flag). Using the event rather than a
-- timer means it fires on the real swap, and the target comes from the server instead
-- of being guessed.
--
-- The camera is turned, not only the body: ToolController.GetTarget picks whoever
-- projects closest to your mouse point on screen, so the camera is what decides who
-- your next attack lands on.
do
	local RunService = game:GetService("RunService")
	local Players    = game:GetService("Players")
	local RSt        = game:GetService("ReplicatedStorage")
	local lp         = Players.LocalPlayer

	ensure("TodoState", { AutoTurn = false, AutoTurnHold = 0.15 })

	local conn = nil

	local function myChar()
		local chars = workspace:FindFirstChild("Characters")
		return (chars and chars:FindFirstChild(lp.Name)) or lp.Character
	end

	-- Never steer while the game owns your transform. Writing the camera during a
	-- hold-to-aim skill drags the aim somewhere else, and writing HumanoidRootPart
	-- while the server is applying knockback makes the two fight and flings you.
	-- Info carries InSkill / Stun for exactly these windows.
	local function safeToSteer(char)
		local info = char and char:FindFirstChild("Info")
		if info and (info:FindFirstChild("InSkill") or info:FindFirstChild("Stun")) then return false end
		if char and (tonumber(char:GetAttribute("Ragdoll")) or 0) > 0 then return false end
		return true
	end

	local function faceTarget(target)
		local char = myChar()
		local hrp = char and char:FindFirstChild("HumanoidRootPart")
		local th  = target and (target:FindFirstChild("HumanoidRootPart") or target.PrimaryPart)
		if not (hrp and th) then return false end
		if not safeToSteer(char) then return false end
		pcall(function()
			local cam = workspace.CurrentCamera
			if cam then cam.CFrame = CFrame.new(cam.CFrame.Position, th.Position) end
			local p = hrp.Position
			-- Rotation only, position preserved, and flattened on Y so you do not tilt
			-- when the target is above or below.
			local flat = Vector3.new(th.Position.X, p.Y, th.Position.Z)
			if (flat - p).Magnitude > 0.1 then hrp.CFrame = CFrame.new(p, flat) end
		end)
		return true
	end

	function _G.SetAutoTurn(on)
		ensure("TodoState", { AutoTurn = false, AutoTurnHold = 0.15 })
		_G.TodoState.AutoTurn = on == true
		if conn then pcall(function() conn:Disconnect() end); conn = nil end
		if not _G.TodoState.AutoTurn then return end

		pcall(function()
			conn = RSt.Knit.Knit.Services.TodoService.RE.Effects.OnClientEvent:Connect(function(name, swapper, target)
				if name ~= "Swap" then return end
				-- Effects is broadcast, so other players' swaps arrive here too.
				if swapper ~= myChar() then return end
				if typeof(target) ~= "Instance" then return end

				task.spawn(function()
					if not faceTarget(target) then return end
					-- One snap is overwritten by the next camera update, so hold the aim
					-- briefly. Kept short so it reads as a flick rather than taking the
					-- camera away from you.
					local hold = math.clamp(tonumber(_G.TodoState.AutoTurnHold) or 0.15, 0, 0.5)
					local until_ = tick() + hold
					local c
					c = RunService.RenderStepped:Connect(function()
						if tick() > until_ or not target.Parent then c:Disconnect() return end
						faceTarget(target)
					end)
				end)
			end)
		end)
	end
end
-- ===================== /Auto Turn 180 on Swap =====================

-- ===================== Higuruma Vote Viewer =====================
-- A draggable readout of the Deadly Sentencing vote counts, taken from the domain's
-- own attributes: workspace.Domains.Domain carries ConfessCount / DenialCount /
-- SilenceCount, which the game also uses to label the three buttons.
--
-- Replaces the earlier over-the-head choice ESP.
--
-- One caveat worth knowing: while testing, the domain parts that actually appeared
-- carried only generic fields (CFrame, Duration, Fade, Health, ...) and no counters,
-- so the panel can legitimately sit at 0 / 0 / 0. If that happens the counts are not
-- being replicated in that situation rather than the panel being broken — the choice
-- is still readable from the line spoken over the accused's head, which can be wired
-- into this same panel.
do
	local RunService = game:GetService("RunService")
	local Players    = game:GetService("Players")
	local lp         = Players.LocalPlayer

	ensure("MiscState", { HiromiVotes = false })

	local GUI_NAME = "SosyHigurumaVotes"
	local conn, attrConn, gui = nil, nil, nil

	local function destroyGui()
		if attrConn then pcall(function() attrConn:Disconnect() end); attrConn = nil end
		if gui then pcall(function() gui:Destroy() end); gui = nil end
		for _, root in ipairs({ (gethui and gethui()) or nil, game:GetService("CoreGui"), lp:FindFirstChild("PlayerGui") }) do
			if root then
				local old = root:FindFirstChild(GUI_NAME)
				if old then pcall(function() old:Destroy() end) end
			end
		end
	end

	local function buildGui()
		local sg = Instance.new("ScreenGui")
		sg.Name = GUI_NAME
		sg.ResetOnSpawn = false
		sg.DisplayOrder = 50
		local ok = pcall(function() sg.Parent = (gethui and gethui()) or game:GetService("CoreGui") end)
		if not ok then sg.Parent = lp:WaitForChild("PlayerGui") end

		local main = Instance.new("Frame")
		main.Name = "Main"
		main.Size = UDim2.fromOffset(250, 44)
		main.Position = UDim2.new(0.5, -125, 0.1, 0)
		main.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
		main.BackgroundTransparency = 0.15
		main.BorderSizePixel = 0
		main.Active = true
		main.Draggable = true
		main.Parent = sg

		local corner = Instance.new("UICorner")
		corner.CornerRadius = UDim.new(0, 6)
		corner.Parent = main

		local stroke = Instance.new("UIStroke")
		stroke.Color = Color3.fromRGB(218, 165, 32)
		stroke.Thickness = 1.2
		stroke.Transparency = 0.3
		stroke.Parent = main

		local title = Instance.new("TextLabel")
		title.Size = UDim2.new(1, 0, 0.5, 0)
		title.BackgroundTransparency = 1
		title.Text = "COURT VOTES"
		title.TextColor3 = Color3.fromRGB(218, 165, 32)
		title.TextSize = 12
		title.Font = Enum.Font.GothamBold
		title.Parent = main

		local votes = Instance.new("TextLabel")
		votes.Name = "Votes"
		votes.Size = UDim2.new(1, 0, 0.5, 0)
		votes.Position = UDim2.new(0, 0, 0.5, 0)
		votes.BackgroundTransparency = 1
		votes.Text = "Domain bekleniyor..."
		votes.TextColor3 = Color3.fromRGB(240, 240, 245)
		votes.TextSize = 11
		votes.Font = Enum.Font.GothamMedium
		votes.Parent = main

		return sg, votes
	end

	function _G.SetHiromiVotes(on)
		ensure("MiscState", { HiromiVotes = false })
		_G.MiscState.HiromiVotes = on == true
		if conn then pcall(function() conn:Disconnect() end); conn = nil end
		destroyGui()
		if not _G.MiscState.HiromiVotes then return end

		local votesLabel
		gui, votesLabel = buildGui()

		local function update(domain)
			if not votesLabel or not votesLabel.Parent then return end
			if not domain then
				votesLabel.Text = "Domain bekleniyor..."
				return
			end
			votesLabel.Text = string.format("Confess: %d  |  Denial: %d  |  Silence: %d",
				tonumber(domain:GetAttribute("ConfessCount")) or 0,
				tonumber(domain:GetAttribute("DenialCount")) or 0,
				tonumber(domain:GetAttribute("SilenceCount")) or 0)
		end

		local lastDomain = nil
		local function hook(domain)
			if attrConn then pcall(function() attrConn:Disconnect() end); attrConn = nil end
			update(domain)
			if domain then
				attrConn = domain.AttributeChanged:Connect(function() update(domain) end)
			end
		end

		conn = RunService.Heartbeat:Connect(function()
			if not (_G.MiscState and _G.MiscState.HiromiVotes) then return end
			local domains = workspace:FindFirstChild("Domains")
			local domain = domains and domains:FindFirstChild("Domain")
			if domain ~= lastDomain then
				lastDomain = domain
				hook(domain)
			end
		end)
	end
end
-- ===================== /Higuruma Vote Viewer =====================

-- ===================== Inf Ult (rapid TP burst at ult end) =====================
-- Fires the moment an awakening actually ends rather than leading into it: the
-- instant InUlt drops, the hub's own Rapid TP feature is switched on for a fixed
-- window and then switched back off.
--
-- Nothing here is character specific -- it only reads InUlt, which every moveset's
-- awakening sets, so it covers all of them.
--
-- If Rapid TP is already running because you turned the toggle on yourself, the
-- burst stays out of the way entirely instead of switching it off under you.
do
	local RunService = game:GetService("RunService")
	local Players    = game:GetService("Players")
	local lp         = Players.LocalPlayer
	ensure("MiscState", { InfUlt = false, InfUltBurst = 5 })

	local conn       = nil
	local bursting   = false
	local burstToken = 0   -- a newer burst (or a cancel) invalidates an in-flight timer

	local function myChar()
		local chars = workspace:FindFirstChild("Characters")
		return (chars and chars:FindFirstChild(lp.Name)) or lp.Character
	end

	local function rapidTp(on)
		if type(_G.SosyHUBStartRapidTP) == "function" then
			pcall(_G.SosyHUBStartRapidTP, on == true)
		end
	end

	local function stopBurst()
		burstToken = burstToken + 1
		if bursting then
			bursting = false
			rapidTp(false)
		end
	end

	local function startBurst()
		if bursting then return end
		-- Already on by the user's own toggle: leave it alone, it is not ours to stop.
		if _G.SosyHUBRapidTPState and _G.SosyHUBRapidTPState.Running then return end

		bursting   = true
		burstToken = burstToken + 1
		local token = burstToken
		local secs  = math.clamp(tonumber(_G.MiscState and _G.MiscState.InfUltBurst) or 5, 1, 15)
		rapidTp(true)
		task.delay(secs, function()
			if token ~= burstToken then return end
			bursting = false
			rapidTp(false)
		end)
	end

	function _G.SetInfUlt(on)
		ensure("MiscState", { InfUlt = false, InfUltBurst = 5 })
		_G.MiscState.InfUlt = on == true
		if conn then pcall(function() conn:Disconnect() end); conn = nil end
		stopBurst()
		if not _G.MiscState.InfUlt then return end

		local wasInUlt = false

		conn = RunService.Heartbeat:Connect(function()
			if not (_G.MiscState and _G.MiscState.InfUlt) then stopBurst() return end
			local char = myChar()
			if not char then wasInUlt = false return end

			local inUlt = char:GetAttribute("InUlt") and true or false
			-- the falling edge is the end of the awakening
			if wasInUlt and not inUlt then startBurst() end
			wasInUlt = inUlt
		end)
	end
end
-- ===================== /Inf Ult (rapid TP burst at ult end) =====================


-- ===================== Instant Blackhole (Yuki) =====================
do
	local RS      = game:GetService("RunService")
	local Players = game:GetService("Players")
	local lp      = Players.LocalPlayer
	ensure("SpecialsState", { InstantBlackhole = false })

	local _instantBhConn = nil

	local function getChar()
		local chars = workspace:FindFirstChild("Characters")
		if chars then
			local c = chars:FindFirstChild(lp.Name)
			if c then return c end
		end
		return lp.Character
	end

	local function isYuki(char)
		if not char then return false end
		local ms = char:GetAttribute("Moveset")
		return ms == "Yuki" or ms == "yuki"
	end

	local BH_KEYWORDS = { "black", "hole", "sacrilegious", "garuda" }
	local function isBlackholeTool(tool)
		local nameL = string.lower(tool.Name)
		local tagL  = string.lower(tostring(tool:GetAttribute("Tag") or ""))
		for _, kw in ipairs(BH_KEYWORDS) do
			if nameL:find(kw, 1, true) or tagL:find(kw, 1, true) then return true end
		end
		return false
	end

	local function zeroBlackholeCooldown(char)
		local moveset = char:FindFirstChild("Moveset")
		if not moveset then return end
		local now = workspace:GetServerTimeNow()
		for _, tool in ipairs(moveset:GetChildren()) do
			if isBlackholeTool(tool) then
				pcall(function()
					local readyAt = tool:GetAttribute("ReadyAt")
					if type(readyAt) == "number" and readyAt > now then
						tool:SetAttribute("ReadyAt", now - 1)
					end
					if tool:GetAttribute("InUse") == true then
						tool:SetAttribute("InUse", false)
					end
				end)
			end
		end
		-- global CD clear attempt
		pcall(function()
			local info = char:FindFirstChild("Info")
			if info and info:GetAttribute("CD") ~= nil then
				info:SetAttribute("CD", true)
				task.delay(0.1, function()
					if info and info.Parent then
						info:SetAttribute("CD", nil)
					end
				end)
			end
		end)
	end

	function _G.SetInstantBlackhole(on)
		ensure("SpecialsState", { InstantBlackhole = false })
		_G.SpecialsState.InstantBlackhole = on == true
		if _instantBhConn then pcall(function() _instantBhConn:Disconnect() end); _instantBhConn = nil end
		if not _G.SpecialsState.InstantBlackhole then return end
		local acc = 0
		_instantBhConn = RS.Heartbeat:Connect(function(dt)
			if not (_G.SpecialsState and _G.SpecialsState.InstantBlackhole) then return end
			acc = acc + dt
			if acc < 0.05 then return end
			acc = 0
			local char = getChar()
			if not char or not isYuki(char) then return end
			pcall(zeroBlackholeCooldown, char)
		end)
	end
end
-- ===================== /Instant Blackhole (Yuki) =====================

-- ===================== Cooldown Viewer =====================
do
	local RS      = game:GetService("RunService")
	local Players = game:GetService("Players")
	local lp      = Players.LocalPlayer
	ensure("MiscState", { CooldownViewer = false })

	local _cdGui      = nil
	local _cdConn     = nil
	local _cdCharConn = nil
	local _cdLabels   = {}

	local function getChar()
		local chars = workspace:FindFirstChild("Characters")
		if chars then
			local c = chars:FindFirstChild(lp.Name)
			if c then return c end
		end
		return lp.Character
	end

	local function getRemainingCd(tool)
		local readyAt = tool:GetAttribute("ReadyAt")
		if type(readyAt) ~= "number" then return 0 end
		local remaining = readyAt - workspace:GetServerTimeNow()
		return math.max(0, remaining)
	end

	local function destroyCdGui()
		if _cdGui then pcall(function() _cdGui:Destroy() end); _cdGui = nil end
		_cdLabels = {}
	end

	local function buildCdGui()
		destroyCdGui()
		pcall(function()
			local hgui  = (gethui and gethui()) or game:GetService("CoreGui")
			local sg    = Instance.new("ScreenGui")
			sg.Name     = "SosyCooldownViewer"
			sg.ResetOnSpawn = false
			sg.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
			sg.Parent   = hgui

			local frame = Instance.new("Frame", sg)
			frame.Name  = "CDFrame"
			frame.Size  = UDim2.new(0, 200, 0, 22)
			frame.Position = UDim2.new(0, 8, 0.5, -50)
			frame.BackgroundColor3 = Color3.fromRGB(14, 14, 20)
			frame.BackgroundTransparency = 0.15
			frame.BorderSizePixel = 0
			local fCorner = Instance.new("UICorner", frame)
			fCorner.CornerRadius = UDim.new(0, 7)
			local fPad = Instance.new("UIPadding", frame)
			fPad.PaddingTop    = UDim.new(0, 3)
			fPad.PaddingBottom = UDim.new(0, 3)
			fPad.PaddingLeft   = UDim.new(0, 6)
			fPad.PaddingRight  = UDim.new(0, 6)
			local fList = Instance.new("UIListLayout", frame)
			fList.SortOrder = Enum.SortOrder.LayoutOrder
			fList.Padding   = UDim.new(0, 2)

			local title = Instance.new("TextLabel", frame)
			title.Name  = "Title"
			title.Size  = UDim2.new(1, 0, 0, 14)
			title.BackgroundTransparency = 1
			title.Text  = "▸ COOLDOWNS"
			title.Font  = Enum.Font.GothamBold
			title.TextSize = 10
			title.TextColor3 = Color3.fromRGB(160, 100, 255)
			title.TextXAlignment = Enum.TextXAlignment.Left
			title.LayoutOrder = 0

			_cdGui = sg
		end)
	end

	local function syncLabels()
		if not _cdGui then return end
		local frame = _cdGui:FindFirstChild("CDFrame")
		if not frame then return end

		local char = getChar()
		local moveset = char and char:FindFirstChild("Moveset")

		local tools = {}
		if moveset then
			for _, t in ipairs(moveset:GetChildren()) do
				local cd = getRemainingCd(t)
				if cd > 0 then
					local name = tostring(t:GetAttribute("Tag") or t.Name)
					table.insert(tools, { name = name, cd = cd })
				end
			end
		end

		table.sort(tools, function(a, b) return a.cd > b.cd end)

		-- reuse existing labels, destroy extras
		local existing = {}
		for _, c in ipairs(frame:GetChildren()) do
			if c:IsA("TextLabel") and c.Name:sub(1, 3) == "cd_" then
				table.insert(existing, c)
			end
		end

		for i, data in ipairs(tools) do
			local row = existing[i]
			if not row then
				row = Instance.new("TextLabel", frame)
				row.Name = "cd_" .. i
				row.Size = UDim2.new(1, 0, 0, 14)
				row.BackgroundColor3 = Color3.fromRGB(30, 20, 50)
				row.BackgroundTransparency = 0.4
				row.BorderSizePixel = 0
				row.Font = Enum.Font.Gotham
				row.TextSize = 10
				row.TextColor3 = Color3.fromRGB(210, 180, 255)
				row.TextXAlignment = Enum.TextXAlignment.Left
				row.LayoutOrder = i
				local c2 = Instance.new("UICorner", row); c2.CornerRadius = UDim.new(0, 4)
			end
			row.Text = string.format(" %-15s  %.1fs", string.sub(data.name, 1, 15), data.cd)
			row.LayoutOrder = i
			row.Visible = true
		end

		for i = #tools + 1, #existing do
			existing[i].Visible = false
		end

		local rowCount = math.max(1, #tools)
		frame.Size = UDim2.new(0, 200, 0, 20 + rowCount * 16)
	end

	function _G.SetCooldownViewer(on)
		ensure("MiscState", { CooldownViewer = false })
		_G.MiscState.CooldownViewer = on == true
		if _cdConn     then pcall(function() _cdConn:Disconnect()     end); _cdConn     = nil end
		if _cdCharConn then pcall(function() _cdCharConn:Disconnect() end); _cdCharConn = nil end
		if not _G.MiscState.CooldownViewer then destroyCdGui(); return end
		buildCdGui()
		local acc = 0
		_cdConn = RS.Heartbeat:Connect(function(dt)
			if not (_G.MiscState and _G.MiscState.CooldownViewer) then return end
			acc = acc + dt
			if acc < 0.12 then return end
			acc = 0
			pcall(syncLabels)
		end)
		_cdCharConn = lp.CharacterAdded:Connect(function()
			if not (_G.MiscState and _G.MiscState.CooldownViewer) then return end
			task.wait(2)
			if _G.MiscState.CooldownViewer then buildCdGui() end
		end)
	end
end
-- ===================== /Cooldown Viewer =====================

_G.RyuState = _G.RyuState or { HideHeatBar = true, SilentVent = true, LocalHeatBar = true }
pcall(hookRyuHeat)
pcall(ensureRyuHeatLoop)
pcall(ensureRyuSilentVentLoop)


]====],
    FeatureMountStandalone = [====[
--[[ SosyHUB FeatureMountStandalone — inventory → SoftUI host, SosyBackend callbacks ]]
return function(win, library)
	local Backend = _G.SosyBackend
	assert(Backend, "SosyBackend missing")
	local used = {}
	local tabSecs = {}
	local tabOrder = 20
	local TAB_ICONS = {
		Blackflash = "lucide:zap", ["Auto Block"] = "lucide:shield", Lock = "lucide:crosshair",
		["Kill All"] = "lucide:skull", Stand = "lucide:user-plus", ["Dash Assist"] = "lucide:move-horizontal",
		Character = "lucide:user", Teleports = "lucide:map-pin", ["Block Break"] = "lucide:swords",
		["Character Specials"] = "lucide:star", Misc = "lucide:box",
		Spoofer = "lucide:fingerprint",
		Shaders = "lucide:sun", Piano = "lucide:music", AI = "lucide:bot",
	}

local TELEPORT_LOCATIONS = {
	"AFK Zone", "Bowling Alley", "Cafeteria", "City", "Domain", "Fight Club",
	"Gym", "Hospital", "Jujutsu High", "Mall", "Parking Lot", "Shibuya",
	"Storage House", "Towers", "Train Button", "Train Station", "Train Station Exit",
	"Tze's", "Under the Map", "Unlicensed Studios",
}

-- ============================================================
-- Embedded Lists
-- These are script-side lists. They are NOT loaded from the user's PC.
-- Add/remove names here to control what appears in the Config List.
-- ============================================================
local CONFIG_LIST = {
    "default",
    "Legit",
    "PvP",
    "Combat",
    "Movement",
    "Visual",
    "Shaders",
    "Piano",
}

local TELEPORT_LIST = TELEPORT_LOCATIONS or {}
local ITEM_LIST = { "None" }
local SAVED_LOCATION_LIST = { "None" }
local SONG_LIST = { "None" }
local TARGET_MODE_LIST = { "Closest", "Closest to Mouse" }
local TARGET_PART_LIST = { "HumanoidRootPart", "Head", "Torso" }
local LOCK_METHOD_LIST = { "Camera", "Body" }

local CATALOG = {
	{
		tab = "Blackflash",
		sec = "BFC",
		side = "left",
		items = {
			{ t = "toggle", n = "Black Flash Chain" },
			{ t = "dropdown", n = "Input Mode", cur = "Both", opts = { "Both", "Keyboard", "Mobile" } },
			{ t = "toggle", n = "BF Mobile Button" },
		},
	},
	{
		tab = "Blackflash",
		sec = "Configuration",
		side = "right",
		items = {
			{ t = "slider", n = "Detection Range", min = 5, max = 80, cur = 50 },
			{ t = "slider", n = "BF Curve Strength", min = 0, max = 40, cur = 12 },
			{ t = "slider", n = "Behind Radius (Distance)", min = 1, max = 12, cur = 5 },
		},
	},
	{
		tab = "Combat",
		sec = "Main Controls",
		side = "left",
		items = {
			{ t = "toggle", n = "Enable Trail Block (M1s)" },
			{ t = "toggle", n = "Enable Move Anim Block" },
			{ t = "toggle", n = "Enable Projectile Block" },
			{ t = "toggle", n = "Block While Attacking" },
		},
	},
	{
		tab = "Combat",
		sec = "Configuration",
		side = "right",
		items = {
			{ t = "slider", n = "Trail Detection Distance", min = 1, max = 40, cur = 13 },
			{ t = "slider", n = "Trail Ahead Angle", min = 0, max = 180, cur = 75 },
			{ t = "slider", n = "Move Anim Detection Range", min = 1, max = 40, cur = 12 },
			{ t = "slider", n = "Move Facing Sensitivity", min = 0, max = 1, cur = 0.7 },
			{ t = "slider", n = "Projectile Detection Range", min = 1, max = 60, cur = 25 },
			{ t = "slider", n = "Block Duration (s)", min = 0, max = 2, cur = 0.5 },
		},
	},
	{
		tab = "Combat",
		sec = "Lock",
		side = "left",
		items = {
			{ t = "toggle", n = "Enable Lock" },
			{ t = "toggle", n = "Mobile Lock Button" },
			{ t = "toggle", n = "Lock ESP" },
			{ t = "toggle", n = "Sticky Target (Until Dead)" },
			{ t = "dropdown", n = "Method", cur = "Camera", opts = LOCK_METHOD_LIST },
			{ t = "dropdown", n = "Target Mode", cur = "Closest", opts = TARGET_MODE_LIST },
			{ t = "dropdown", n = "Target Part", cur = "HumanoidRootPart", opts = TARGET_PART_LIST },
			{ t = "slider", n = "Smoothness", min = 0, max = 1, cur = 0 },
			{ t = "slider", n = "Camera Side Offset", min = 0, max = 10, cur = 2 },
		},
	},
	{
		tab = "Combat",
		sec = "Actions",
		side = "right",
		items = {
			{ t = "toggle", n = "Exclude Domains" },
			{ t = "toggle", n = "Exclude Players" },
			{ t = "dropdown", n = "Exclude Players List", cur = "None", opts = { "None", "Dummy" } },
			{ t = "slider", n = "TP Stay Time", min = 0, max = 1, cur = 0.05 },
			{ t = "button", n = "Refresh Player List" },
		},
	},
	{
		tab = "Combat",
		sec = "SosyHUB",
		side = "right",
		items = {
			{ t = "toggle", n = "SosyHUB AC Bypass" },
			{ t = "toggle", n = "SosyHUB Rapid TP" },
			{ t = "status", n = "Status: SosyHUB AC Idle" },
		},
	},
	{
		tab = "Character",
		sec = "Main Target",
		side = "left",
		items = {
			{ t = "dropdown", n = "Select Target", cur = "Dummy", opts = { "Dummy", "None" } },
			{ t = "toggle", n = "Attach" },
			{ t = "toggle", n = "Attach on Sound" },
			{ t = "toggle", n = "Main Target ESP" },
			{ t = "toggle", n = "Spectate Main Target" },
			{ t = "button", n = "Refresh Targets" },
		},
	},
	{
		tab = "Character",
		sec = "Utility",
		side = "right",
		items = {
			{ t = "dropdown", n = "Select Utility Target", cur = "Dummy", opts = { "Dummy", "None" } },
			{ t = "toggle", n = "Abuse Target" },
			{ t = "toggle", n = "Aura (auto M1 nearby)" },
			{ t = "toggle", n = "Auto Soda On Low Health" },
			{ t = "toggle", n = "Utility Target ESP" },
			{ t = "toggle", n = "Spectate Utility Target" },
			{ t = "button", n = "Refresh Utility Targets" },
			{ t = "button", n = "Bring to User's Front" },
		},
	},
	{
		tab = "Movement",
		sec = "Side Dash Settings",
		side = "left",
		items = {
			{ t = "toggle", n = "Enable Side Dash Assist" },
			{ t = "toggle", n = "Lock Camera On Enemy" },
			{ t = "toggle", n = "Dash Only If Facing Front" },
			{ t = "slider", n = "Detection Distance", min = 5, max = 100, cur = 60 },
			{ t = "slider", n = "Behind Distance", min = 1, max = 15, cur = 5 },
			{ t = "slider", n = "Flight Duration", min = 0.05, max = 2, cur = 0.42 },
			{ t = "toggle", n = "Enable Mobile Button" },
		},
	},
	{
		tab = "Movement",
		sec = "Arc Modifiers",
		side = "right",
		items = {
			{ t = "slider", n = "Curve Strength", min = 0, max = 40, cur = 10 },
			{ t = "slider", n = "Arch Height", min = 0, max = 15, cur = 3 },
			{ t = "slider", n = "Lock Duration", min = 0, max = 2, cur = 0.35 },
		},
	},
	{
		tab = "Movement",
		sec = "Teleportation",
		side = "left",
		items = {
			{ t = "list", n = "Location", cur = "Bowling Alley", opts = TELEPORT_LIST },
			{ t = "button", n = "Teleport" },
		},
	},
	{
		tab = "Movement",
		sec = "Item Grabber",
		side = "left",
		items = {
			{ t = "list", n = "Select Item", cur = "None", opts = ITEM_LIST },
			{ t = "button", n = "Refresh Items" },
			{ t = "button", n = "Grab Item" },
		},
	},
	{
		tab = "Movement",
		sec = "Save Location",
		side = "right",
		items = {
			{ t = "list", n = "Saved Locations", cur = "None", opts = SAVED_LOCATION_LIST },
			{ t = "button", n = "Save Current Position" },
			{ t = "button", n = "Go to Saved Location" },
		},
	},
	{
		tab = "Combat",
		sec = "Main Controls",
		side = "left",
		items = {
			{ t = "toggle", n = "Enable Block Break" },
			{ t = "toggle", n = "Lock Behind Enemy" },
		},
	},
	{
		tab = "Combat",
		sec = "Configuration",
		side = "right",
		items = {
			{ t = "slider", n = "BB Detection Distance", min = 1, max = 40, cur = 15 },
			{ t = "slider", n = "Behind Radius", min = 1, max = 15, cur = 4.5 },
			{ t = "slider", n = "Dash Duration (s)", min = 0.05, max = 1, cur = 0.12 },
			{ t = "slider", n = "BB Curve Strength", min = 0, max = 30, cur = 8 },
		},
	},
	{
		tab = "Character",
		sec = "Nanami",
		side = "left",
		items = {
			{ t = "toggle", n = "Auto Ratio" },
			{ t = "toggle", n = "Auto Final Judgement" },
			{ t = "slider", n = "Auto Skill Range", min = 10, max = 150, cur = 60 },
		},
	},
	{
		tab = "Character",
		sec = "Yuki",
		side = "left",
		items = {
			{ t = "toggle", n = "Garuda Rebound" },
			{ t = "slider", n = "Garuda Delay", min = 0, max = 3, cur = 1 },
		},
	},
	{
		tab = "Character",
		sec = "Yuji",
		side = "left",
		items = {
			{ t = "toggle", n = "Auto WCS" },
		},
	},
	{
		tab = "Character",
		sec = "Todo",
		side = "right",
		items = {
			{ t = "toggle", n = "Auto Perfect Clap" },
			{ t = "toggle", n = "Auto Turn 180 on Swap" },
			{ t = "slider", n = "Auto Turn Hold (ms)", min = 0, max = 500, cur = 150 },
			{ t = "slider", n = "Perfect Clap Look (ms)", min = 0, max = 400, cur = 150 },
			{ t = "slider", n = "Perfect Clap Delay (ms)", min = 0, max = 300, cur = 50 },
			{ t = "toggle", n = "Fast Swap (Predict)" },
			{ t = "slider", n = "Fast Swap Offset", min = 0, max = 6, cur = 1.5 },
			{ t = "toggle", n = "Fast Clap Anim (visual)" },
			{ t = "slider", n = "Clap Anim Speed", min = 1, max = 8, cur = 3 },
			{ t = "dropdown", n = "Todo Bring Target", cur = "Dummy", opts = { "Dummy", "None" } },
			{ t = "slider", n = "Todo Bring Range", min = 10, max = 120, cur = 50 },
			{ t = "button", n = "Refresh Targets" },
			{ t = "button", n = "Todo Bring" },
		},
	},
	{
		tab = "Character",
		sec = "Naoya",
		side = "right",
		items = {
			{ t = "toggle", n = "Naoya Infinite Front Dash" },
		},
	},
	{
		tab = "Character",
		sec = "Higuruma",
		side = "right",
		items = {
			{ t = "toggle", n = "Auto QTE (Final Judgement)" },
			{ t = "toggle", n = "Auto Vote (Domain)" },
		},
	},
	{
		tab = "Character",
		sec = "Hakari",
		side = "right",
		items = {
			{ t = "toggle", n = "Auto Door - Hakari" },
			{ t = "toggle", n = "Fever Crusher" },
		},
	},
	{
		tab = "Character",
		sec = "Mahito",
		side = "right",
		items = {
			{ t = "toggle", n = "Auto Body Repel" },
		},
	},
	{
		tab = "Character",
		sec = "Auto Blackflashes",
		side = "left",
		items = {
			{ t = "toggle", n = "Heian Sukuna Blackflash" },
			{ t = "slider", n = "Heian Cleave Delay", min = 0, max = 1, cur = 0.2 },
			{ t = "toggle", n = "Mahito Blackflash" },
			{ t = "toggle", n = "Todo Blackflash" },
			{ t = "toggle", n = "Yuta Blackflash" },
		},
	},
	{
		tab = "Misc",
		sec = "Misc",
		side = "left",
		items = {
			{ t = "toggle", n = "Invisible" },
			{ t = "toggle", n = "Infinite Dash" },
			{ t = "toggle", n = "Unlock Extra Emote Slot" },
			{ t = "toggle", n = "Noclip Domain" },
			{ t = "toggle", n = "Infinite Parkour" },
			{ t = "toggle", n = "Inf Ult" },
			{ t = "slider", n = "Inf Ult TP Duration (s)", min = 1, max = 15, cur = 5 },
			{ t = "toggle", n = "Higuruma Vote Viewer" },
			{ t = "toggle", n = "Anti Domain" },
			{ t = "toggle", n = "Cooldown Viewer" },
			{ t = "button", n = "Force Reset" },
			{ t = "button", n = "Call Train" },
			{ t = "button", n = "Old Ladder" },
		},
	},
	{
		tab = "Misc",
		sec = "Movement",
		side = "right",
		items = {
			{ t = "toggle", n = "Flight" },
			{ t = "slider", n = "Base Speed", min = 16, max = 300, cur = 150 },
			{ t = "slider", n = "Boost Speed (Shift)", min = 16, max = 500, cur = 300 },
			{ t = "slider", n = "Max FOV", min = 70, max = 120, cur = 120 },
			{ t = "slider", n = "Camera FOV", min = 40, max = 120, cur = 70 },
		},
	},
	{
		tab = "Misc",
		sec = "avatar spoofer",
		side = "left",
		items = {
			{ t = "textbox", n = "Avatar Username", placeholder = "Roblox username to spoof" },
			{ t = "button", n = "apply avatar" },
		},
	},
	{
		tab = "Misc",
		sec = "fps spoofer",
		side = "right",
		items = {
			{ t = "toggle", n = "enable fps spoof" },
			{ t = "slider", n = "Minimum FPS", min = 1, max = 240, cur = 30 },
			{ t = "slider", n = "Maximum FPS", min = 1, max = 240, cur = 60 },
		},
	},
	{
		tab = "UI Settings",
		sec = "Config Manager",
		side = "left",
		items = {
			{ t = "textbox", n = "Config Name", cur = "", placeholder = "Enter config name..." },
			{ t = "list", n = "Saved Configs", cur = "default", opts = CONFIG_LIST },
			{ t = "button", n = "Save Config" },
			{ t = "button", n = "Load Config" },
			{ t = "button", n = "Delete Config" },
			{ t = "button", n = "Refresh Config List" },
		},
	},
	{
		tab = "UI Settings",
		sec = "Auto Load",
		side = "right",
		items = {
			{ t = "dropdown", n = "Auto Load Config", cur = "None", opts = { "None" } },
			{ t = "toggle", n = "Enable Auto Load" },
		},
	},
	{
		tab = "Character",
		sec = "Character",
		side = "left",
		items = {
			{ t = "toggle", n = "Anti-Stun / Knockback" },
			{ t = "toggle", n = "Anti-Ragdoll" },
			{ t = "toggle", n = "Hitbox Expander" },
			{ t = "slider", n = "Hitbox Size", min = 1, max = 100, cur = 5 },
			{ t = "toggle", n = "No Dash Cooldowns" },
			{ t = "slider", n = "Dash Reset Rate (ms)", min = 20, max = 1000, cur = 100 },
			{ t = "toggle", n = "Hitbox Range" },
			{ t = "slider", n = "Hitbox Range Mult", min = 1, max = 12, cur = 3 },
			{ t = "slider", n = "Hitbox Max Radius", min = 5, max = 120, cur = 30 },
		},
	},
	{
		tab = "Character",
		sec = "Yuki Extra",
		side = "left",
		items = {
			{ t = "toggle", n = "Yuki Instant Charge" },
			{ t = "toggle", n = "Instant Blackhole" },
		},
	},
	{
		tab = "Character",
		sec = "Ryu",
		side = "right",
		items = {
			{ t = "toggle", n = "Ryu Silent Vent" },
			{ t = "toggle", n = "Ryu Hide Heat Bar" },
			{ t = "toggle", n = "Ryu Local Heat Bar" },
		},
	},
	{
		tab = "Misc",
		sec = "Misc Extra",
		side = "left",
		items = {
			{ t = "toggle", n = "Auto Buy Soda When Low" },
			{ t = "slider", n = "Soda Buy HP %", min = 10, max = 90, cur = 40 },
			{ t = "toggle", n = "0.2 Domain Farm" },
		},
	},
	{
		tab = "Shaders",
		sec = "Master",
		side = "left",
		items = {
			{ t = "toggle", n = "Shaders Enabled", cur = true },
		},
	},
	{
		tab = "Shaders",
		sec = "Shaders",
		side = "left",
		items = {
			{ t = "button", n = "Default" },
			{ t = "button", n = "Morning" },
			{ t = "button", n = "Midday" },
			{ t = "button", n = "Afternoon" },
			{ t = "button", n = "Evening" },
			{ t = "button", n = "Night" },
			{ t = "button", n = "Midnight" },
		},
	},
	{
		tab = "Shaders",
		sec = "Shaders Lite",
		side = "right",
		items = {
			{ t = "button", n = "Default Lite" },
			{ t = "button", n = "Morning Lite" },
			{ t = "button", n = "Midday Lite" },
			{ t = "button", n = "Afternoon Lite" },
			{ t = "button", n = "Evening Lite" },
			{ t = "button", n = "Night Lite" },
			{ t = "button", n = "Midnight Lite" },
		},
	},
	{
		tab = "Shaders",
		sec = "Weather",
		side = "left",
		items = {
			{ t = "button", n = "Default Weather" },
			{ t = "button", n = "Rain" },
			{ t = "button", n = "Snow" },
			{ t = "button", n = "Fog" },
			{ t = "button", n = "Sunny" },
			{ t = "button", n = "Cloudy" },
			{ t = "button", n = "Storm" },
		},
	},
	{
		tab = "Shaders",
		sec = "Graphics Settings",
		side = "right",
		items = {
			{ t = "slider", n = "shaders quality", min = 1, max = 21, cur = 21 },
			{ t = "button", n = "copy saved adjustment to clipboard" },
		},
	},
	{
		tab = "Shaders",
		sec = "Time Adjustment",
		side = "right",
		items = {
			{ t = "slider", n = "clock time", min = 0, max = 24, cur = 14 },
			{ t = "slider", n = "geographic latitude", min = 0, max = 180, cur = 0 },
		},
	},
	{
		tab = "Shaders",
		sec = "Clouds Adjustment",
		side = "left",
		items = {
			{ t = "slider", n = "clouds cover", min = 0, max = 1, cur = 0 },
			{ t = "slider", n = "clouds density", min = 0, max = 1, cur = 0 },
		},
	},
	{
		tab = "Shaders",
		sec = "Atmosphere Adjustment",
		side = "right",
		items = {
			{ t = "slider", n = "atmosphere density", min = 0, max = 1, cur = 0.3 },
			{ t = "slider", n = "atmosphere offset", min = 0, max = 1, cur = 0 },
			{ t = "slider", n = "atmosphere glare", min = 0, max = 10, cur = 0 },
			{ t = "slider", n = "atmosphere haze", min = 0, max = 10, cur = 0 },
		},
	},
	{
		tab = "Shaders",
		sec = "Depth Of Field",
		side = "left",
		items = {
			{ t = "toggle", n = "depthoffield enabled", cur = true },
			{ t = "slider", n = "dof farintensity", min = 0, max = 1, cur = 0.1 },
			{ t = "slider", n = "dof focusdistance", min = 0, max = 200, cur = 0.05 },
			{ t = "slider", n = "dof infocusradius", min = 0, max = 50, cur = 10 },
			{ t = "slider", n = "dof nearintensity", min = 0, max = 1, cur = 0.75 },
		},
	},
	{
		tab = "Shaders",
		sec = "Sunrays",
		side = "right",
		items = {
			{ t = "toggle", n = "sunrays enabled", cur = true },
			{ t = "slider", n = "sunrays intensity", min = 0, max = 1, cur = 0.25 },
			{ t = "slider", n = "sunrays spread", min = 0, max = 1, cur = 1 },
		},
	},
	{
		tab = "Shaders",
		sec = "Color Correction",
		side = "left",
		items = {
			{ t = "toggle", n = "colorcor enabled", cur = true },
			{ t = "slider", n = "colorcor brightness", min = -1, max = 1, cur = 0 },
			{ t = "slider", n = "colorcor contrast", min = -1, max = 1, cur = 0 },
			{ t = "slider", n = "colorcor saturation", min = -1, max = 1, cur = 1 },
		},
	},
	{
		tab = "Shaders",
		sec = "Bloom",
		side = "left",
		items = {
			{ t = "toggle", n = "bloom enabled", cur = true },
			{ t = "slider", n = "bloom intensity", min = 0, max = 1, cur = 0.4 },
			{ t = "slider", n = "bloom size", min = 0, max = 56, cur = 24 },
			{ t = "slider", n = "bloom threshold", min = 0, max = 4, cur = 0.95 },
		},
	},

	{
		tab = "Piano",
		sec = "Piano",
		side = "left",
		items = {
			{ t = "toggle", n = "Piano Enabled" },
			{ t = "list", n = "Song", cur = "None", opts = SONG_LIST },
			{ t = "button", n = "Refresh Songs" },
			{ t = "button", n = "Play Song" },
			{ t = "button", n = "Pause Song" },
			{ t = "button", n = "Stop Song" },
		},
	},
	{
		tab = "Piano",
		sec = "Playback",
		side = "right",
		items = {
			{ t = "slider", n = "Piano Speed", min = 0.25, max = 3, cur = 1 },
			{ t = "slider", n = "Piano Transpose", min = -24, max = 24, cur = 0 },
			{ t = "toggle", n = "Piano Loop" },
		},
	},
	{
		tab = "UI Settings",
		sec = "Appearance",
		side = "left",
		items = {
			{ t = "dropdown", n = "Select Theme", cur = "Red/Black", opts = ThemeOrder },
		},
	},
	{
		tab = "UI Settings",
		sec = "Font",
		side = "right",
		items = {
			{ t = "dropdown", n = "UI Font", cur = "GothamBold", opts = { "GothamBold", "Gotham", "SourceSansPro", "RobotoMono", "Oswald", "SciFi", "Fantasy", "Cartoon", "Code", "Bodoni" } },
		},
	},
}

	-- Luxury compact navigation: related features share a single tab.
	-- This also prevents empty / one-feature tabs from cluttering the sidebar.
	-- Keep the useful high-level tabs, but remove one-feature tabs such as
	-- Blackflash. Blackflash belongs with the other character-specific tools.
	local TAB_REMAP = {
		Blackflash = "Character Specials",
		["Auto Block"] = "Auto Block",
		Lock = "Lock",
		["Kill All"] = "Kill All",
		Stand = "Stand",
		["Dash Assist"] = "Dash Assist",
		["Block Break"] = "Block Break",
		Character = "Character",
		Teleports = "Teleports",
		Spoofer = "Spoofer",
		Misc = "Misc",
		["Character Specials"] = "Character Specials",
		Shaders = "Shaders",
		Piano = "Piano",
		AI = "AI",
		["UI Settings"] = "UI Settings",
	}

	for _, block in ipairs(CATALOG) do
		block.tab = TAB_REMAP[block.tab] or block.tab
	end

	local TABS = {
		"Character",
		"Combat",
		"Movement",
		"Misc",
		"Shaders",
		"Piano",
		"AI",
		"UI Settings",
	}

	-- Wire theme dropdown callback before CATALOG is finalized
	pcall(function()
		local savedTheme = _G.SosyLastTheme or "Red/Black"
		if ThemePresets[savedTheme] then applyPreset(savedTheme) end
	end)

	Backend.initDefaults(CATALOG)
	pcall(function()
		local mode = tostring(_G.SosyUIMode or "PC")
		if mode == "Mobile" then
			Backend.set("Shaders Enabled", false)
			if _G.SosyShaders and _G.SosyShaders.applyPlatformDefaults then
				_G.SosyShaders.applyPlatformDefaults("Mobile")
			elseif _G.SosyShaders then
				_G.SosyShaders.applyControl("Shaders Enabled", false)
			end
			return
		end
		Backend.set("Shaders Enabled", true)
		Backend.set("Camera FOV", 70)
		Backend.set("shaders quality", 21)
		Backend.set("sunrays enabled", true)
		Backend.set("colorcor enabled", true)
		Backend.set("colorcor saturation", 1)
		Backend.set("bloom enabled", true)
		Backend.set("depthoffield enabled", true)
		if _G.SosyShaders then
			local S = _G.SosyShaders
			local function applyPc()
				if type(S.applyPlatformDefaults) == "function" then
					S.applyPlatformDefaults("PC")
				else
					S.applyControl("Shaders Enabled", true)
					S.setCameraFov(70)
					S.click("Afternoon Lite")
					S.applyControl("shaders quality", 21)
					S.applyControl("sunrays enabled", true)
					S.applyControl("colorcor enabled", true)
					S.applyControl("colorcor saturation", 1)
				end
			end
			applyPc()
			task.delay(0.8, applyPc)
			task.delay(2.0, applyPc)
		end
	end)
	Backend.refreshConfigList()
	pcall(function() Backend.bootstrapFromVps() end)

	local function ensureTabShell(tabName)
		tabName = tostring(tabName or "Misc")
		if tabSecs[tabName] then return tabSecs[tabName] end
		tabOrder = tabOrder + 1
		local tab = win:Tab({
			name = tabName,
			icon = TAB_ICONS[tabName] or "lucide:layers",
			_layoutOrder = tabOrder,
		})
		tabSecs[tabName] = { tab = tab, byKey = {} }
		return tabSecs[tabName]
	end

	local function ensureTab(tabName, secName, side)
		tabName = tostring(tabName or "Misc")
		-- Luxury layout: feature sections are a single full-width vertical stream.
		-- The old left/right split caused controls to be squeezed into the edge.
		side = (side == "right") and "right" or "left"
		secName = tostring(secName or "Main")
		local meta = ensureTabShell(tabName)
		local key = string.lower(secName) .. "|" .. side
		if meta.byKey[key] then return meta.byKey[key] end
		local sec = meta.tab:Section({ name = secName, side = side })
		meta.byKey[key] = sec
		return sec
	end

	-- Tab shells only (do NOT create empty tab-name sections)
	for _, t in ipairs(TABS) do
		pcall(ensureTabShell, t)
	end

	local function place(sec, it)
		local name = it.n
		if not name or used[name] then return end
		used[name] = true
		local kind = it.t
		if kind == "toggle" then
			local def = Backend.get(name) == true
			local tog = sec:Toggle({
				name = name,
				default = def,
				Callback = function(v)
					Backend.set(name, v == true)
				end,
			})
			Backend.bindUi(name, {
				_uiSet = function(v)
					if tog and tog.set then tog:set(v == true, false) end
				end,
			})
		elseif kind == "slider" then
			local minV, maxV = tonumber(it.min) or 0, tonumber(it.max) or 100
			local cur = tonumber(Backend.get(name)) or tonumber(it.cur) or minV
			sec:Slider({
				name = name,
				min = minV,
				max = maxV,
				default = cur,
				decimals = (maxV - minV) <= 5 and 2 or 0,
				Callback = function(v)
					Backend.set(name, v)
				end,
			})
		elseif kind == "dropdown" or kind == "list" then
			local opts = it.opts or { "None" }
			local cur = tostring(Backend.get(name) or it.cur or opts[1] or "None")
			local maker = (kind == "list" and sec.List) or sec.Dropdown
			local api = maker(sec, {
				name = name,
				items = opts,
				default = cur,
				Callback = function(v)
					Backend.set(name, v)
				end,
			})
			Backend.bindUi(name, {
				_uiSetOptions = function(o)
					if not api then return end
					local list = o or {}
					-- SoftUI / macWin wrapper
					if type(api.set_items) == "function" then
						local ok = pcall(function() api:set_items(list) end)
						if ok then return end
						pcall(function() api.set_items(api, list) end)
						return
					end
					-- Real MacLib dropdown API
					if type(api.ClearOptions) == "function" then pcall(function() api:ClearOptions() end) end
					if type(api.InsertOptions) == "function" then
						pcall(function() api:InsertOptions(list) end)
						return
					end
					for _, m in ipairs({"SetItems","SetValues","SetOptions","SetList","updateItems","Update"}) do
						if type(api[m]) == "function" then
							pcall(function() api[m](api, list) end)
							return
						end
					end
				end,
				_uiSet = function(v)
					if not api then return end
					for _, m in ipairs({"set_value","SetValue","SetSelected","set_selected","Select","SetDefault"}) do
						if type(api[m]) == "function" then
							pcall(function() api[m](api, v) end)
							break
						end
					end
				end,
			})
		elseif kind == "textbox" then
			local cur = tostring(Backend.get(name) or it.cur or "")
			sec:Textbox({
				name = name,
				placeholder = it.placeholder or name,
				default = cur,
				Callback = function(v)
					Backend.set(name, tostring(v or ""))
				end,
			})
		elseif kind == "button" then
			sec:Button({
				name = name,
				Callback = function()
					Backend.click(name)
				end,
			})
		elseif kind == "status" then
			local lbl = sec:Label({ name = tostring(name) })
			Backend.bindUi(name, {
				_uiSet = function(v)
					if lbl and lbl.set then lbl.set(tostring(v)) end
				end,
			})
		end
	end

	-- 207 controls built in one synchronous burst is enough UI construction
	-- (Instance.new + UICorner/UIStroke + tweens, per control) to blow a frame
	-- budget and read as a multi-second stutter. Yield every few controls so
	-- the hub fills in gradually instead of all landing in a single frame.
	local mountBudget = 0
	local function mountBreathe()
		mountBudget = mountBudget + 1
		if mountBudget >= 6 then
			mountBudget = 0
			task.wait()
		end
	end
	local _lastTab, _lastSec = nil, nil
	for _, block in ipairs(CATALOG) do
		if block.tab ~= _lastTab or block.sec ~= _lastSec then
			_lastTab, _lastSec = block.tab, block.sec
		end
		local sec = ensureTab(block.tab, block.sec, block.side)
		for _, it in ipairs(block.items or {}) do
			pcall(place, sec, it)
			mountBreathe()
		end
	end

	-- Full-tab AI chat (Cursor API — no canned replies)
	pcall(function()
		-- DISABLED: under the MacLib wrapper, aiSec.elements is an unparented Frame,
		-- so this SoftUI-style chat never renders and only leaves an empty "Chat"
		-- section in the AI tab. The working AI chat is injected in buildHub below.
		if true then return end
		local aiSec = ensureTab("AI", "Chat", "left")
		local host = aiSec.elements
		host.ClipsDescendants = true
		local history = Instance.new("ScrollingFrame")
		history.Name = "AIChatHistory"
		history.Size = UDim2.new(1, 0, 1, -52)
		history.BackgroundColor3 = Color3.fromRGB(248, 249, 251)
		history.BorderSizePixel = 0
		history.ScrollBarThickness = 4
		history.CanvasSize = UDim2.new()
		history.AutomaticCanvasSize = Enum.AutomaticSize.Y
		history.ZIndex = 7
		history.Parent = host
		history:SetAttribute("SosyTheme", "page")
		local hc = Instance.new("UICorner")
		hc.CornerRadius = UDim.new(0, 10)
		hc.Parent = history
		local hLay = Instance.new("UIListLayout")
		hLay.Padding = UDim.new(0, 6)
		hLay.Parent = history
		local hPad = Instance.new("UIPadding")
		hPad.PaddingTop = UDim.new(0, 8)
		hPad.PaddingBottom = UDim.new(0, 8)
		hPad.PaddingLeft = UDim.new(0, 8)
		hPad.PaddingRight = UDim.new(0, 8)
		hPad.Parent = history
		local chatHist = {}
		local function themeCols()
			local t = _G.SosyThemeColors
			if type(t) == "table" then return t end
			return {
				Page = Color3.fromRGB(248, 249, 251),
				Card = Color3.fromRGB(255, 255, 255),
				Text = Color3.fromRGB(26, 26, 26),
				Muted = Color3.fromRGB(112, 112, 112),
				ActiveBg = Color3.fromRGB(245, 240, 255),
				Accent = Color3.fromRGB(125, 60, 255),
			}
		end
		local function addBubble(who, text)
			local c = themeCols()
			local b = Instance.new("TextLabel")
			b.BackgroundColor3 = who == "you" and c.ActiveBg or c.Card
			b.Size = UDim2.new(1, -4, 0, 0)
			b.AutomaticSize = Enum.AutomaticSize.Y
			b.Font = Enum.Font.Gotham
			b.TextSize = 13
			b.TextColor3 = c.Text
			b.TextXAlignment = Enum.TextXAlignment.Left
			b.TextWrapped = true
			b.Text = (who == "you" and "You: " or "AI: ") .. tostring(text)
			b.ZIndex = 8
			b.Parent = history
			local corner = Instance.new("UICorner")
			corner.CornerRadius = UDim.new(0, 8)
			corner.Parent = b
			local p = Instance.new("UIPadding")
			p.PaddingTop = UDim.new(0, 8)
			p.PaddingBottom = UDim.new(0, 8)
			p.PaddingLeft = UDim.new(0, 10)
			p.PaddingRight = UDim.new(0, 10)
			p.Parent = b
			task.defer(function()
				history.CanvasPosition = Vector2.new(0, history.AbsoluteCanvasSize.Y)
			end)
		end
		addBubble("ai", "SosyHUB AI online. Ask anything.")
		history.Size = UDim2.new(1, 0, 1, -44)
		local row = Instance.new("Frame")
		row.BackgroundTransparency = 1
		row.AnchorPoint = Vector2.new(0.5, 1)
		row.Size = UDim2.fromOffset(340, 26)
		row.Position = UDim2.new(0.5, 0, 1, -6)
		row.ZIndex = 7
		row.Parent = host
		local input = Instance.new("TextBox")
		input.Size = UDim2.new(1, -58, 1, 0)
		input.BackgroundColor3 = Color3.fromRGB(248, 249, 251)
		input.PlaceholderText = "Message AI..."
		input.Text = ""
		input.TextColor3 = Color3.fromRGB(26, 26, 26)
		input.ClearTextOnFocus = false
		input.TextSize = 11
		input.Font = Enum.Font.Gotham
		input.ZIndex = 8
		input.Parent = row
		input:SetAttribute("SosyTheme", "page")
		local ic = Instance.new("UICorner")
		ic.CornerRadius = UDim.new(0, 6)
		ic.Parent = input
		local send = Instance.new("TextButton")
		send.Size = UDim2.new(0, 50, 1, 0)
		send.Position = UDim2.new(1, -50, 0, 0)
		send.BackgroundColor3 = Color3.fromRGB(125, 60, 255)
		send.Text = "Send"
		send.TextColor3 = Color3.fromRGB(255, 255, 255)
		send.TextSize = 11
		send.Font = Enum.Font.Gotham
		send.AutoButtonColor = false
		send.ZIndex = 8
		send.Parent = row
		local sc = Instance.new("UICorner")
		sc.CornerRadius = UDim.new(0, 6)
		sc.Parent = send
		local busy = false
		local function doSend()
			if busy then return end
			local msg = tostring(input.Text or ""):gsub("^%s+", ""):gsub("%s+$", "")
			if msg == "" then return end
			busy = true
			addBubble("you", msg)
			input.Text = ""
			addBubble("ai", "...")
			task.spawn(function()
				local reply, err
				if type(_G.SosyAIAsk) == "function" then
					reply, err = _G.SosyAIAsk(msg, chatHist)
				else
					err = "SosyAIAsk missing"
				end
				-- replace last "..." bubble
				local kids = history:GetChildren()
				for i = #kids, 1, -1 do
					local k = kids[i]
					if k:IsA("TextLabel") and k.Text == "AI: ..." then
						k:Destroy()
						break
					end
				end
				if reply then
					table.insert(chatHist, { role = "user", content = msg })
					table.insert(chatHist, { role = "assistant", content = reply })
					addBubble("ai", reply)
				else
					addBubble("ai", "API error: " .. tostring(err))
				end
				busy = false
			end)
		end
		send.MouseButton1Click:Connect(doSend)
		input.FocusLost:Connect(function(enter)
			if enter then doSend() end
		end)
	end)

	-- Final pass: bind premium micro-interactions to every interactive feature widget.
	pcall(function()
		for _, d in ipairs(featureRoot:GetDescendants()) do
			if d:IsA("TextButton") and not d:GetAttribute("SosyLuxuryButton") then
				setupLuxuryButton(d, d.Text, findCard(d))
			end
		end
	end)

	-- Auto load
	pcall(function()
		if Backend.get("Enable Auto Load") == true then
			local cfg = Backend.get("Auto Load Config")
			if cfg and cfg ~= "None" then Backend.loadConfig(cfg) end
		end
	end)

	_G._SosyNovaFeaturesMounted = true
	_G._SosyFeatureMountStats = {
		tabs = #TABS,
		controls = (function()
			local n = 0
			for _ in pairs(used) do n = n + 1 end
			return n
		end)(),
		inventory = { tabs = #TABS, note = "compact luxury navigation" },
	}
	print(string.format("[SosyHUB] FeatureMountStandalone ok tabs=%d controls=%d", #TABS, _G._SosyFeatureMountStats.controls))
end

]====],
}

-- Cascade UI library from source 2.
local cascade = loadstring([====[
--!nolint
--!nocheck
--!optimize 2

--[[
    @author Cascade UI
    @name Cascade
    @description A Luau UI library based on macOS Sequoia.
    @license MIT License

    @buildDate "2026-05-21T17:25:02.724349900+00:00"
    @buildCfg "Release"
    @buildVers "v1.4.0"

    This file was automatically generated with darklua, it is not intended for manual editing.
--]]

__SH = { COMPINDEXES = {} }

local a={cache={}}do do local b=function()local b={}b.Clone=function(c)local d
if type(c)=='function'then d=clonefunction and clonefunction(c)elseif typeof(c)
=='Instance'then d=cloneref and cloneref(c)end return d or c end b.ProtectUI=
function(c)local d,e=pcall(function()return gethui()end)local f,g=pcall(function
()local f=b.Clone(game:GetService'CoreGui')if f and f.ClassName then return f
end return end)local h,i=pcall(function()c.Parent=d and e or f and g or b.Clone(
game:GetService'Players').LocalPlayer.PlayerGui end)return h and c or i end
return b end function a.a()local c=a.cache.a if not c then c={c=b()}a.cache.a=c
end return c.c end end do local b=function()return 1 end function a.b()local c=a
.cache.b if not c then c={c=b()}a.cache.b=c end return c.c end end do local b=
function()local b={}function b.Apply(c,d,e)for f,g in pairs(c)do if e and table.
find(e,f)then continue end pcall(function()d[f]=g end)end return d end function
b.ReactiveTable(c,d)local e,f=table.clone(c or{}),{}local g={__index=function(g,
h)return e[h]end,__newindex=function(g,h,i)e[h]=i if d then d(e)end end,__len=
function()return#e end,__pairs=function()return pairs(e)end,__ipairs=function()
return ipairs(e)end,__metatable=false}return setmetatable(f,g)end function b.
Wrap(c,d,e,f)local g={}setmetatable(g,{__index=function(h,i)if c[i]~=nil then
return c[i]elseif e then local j,k=pcall(function()return e[i]end)if j then
return k end end return end,__newindex=function(h,i,j)local k=d[i]if k then k(j)
if not(f and table.find(f,i))then c[i]=j end elseif e then local l=pcall(
function()e[i]=j end)if not l then c[i]=j end else c[i]=j end end})return g end
return b end function a.c()local c=a.cache.c if not c then c={c=b()}a.cache.c=c
end return c.c end end do local b=function()local b,c,d=a.b(),a.c(),{}function d
.Value(e)local f={}return c.Wrap({Value=e,Connect=function(g,h)table.insert(f,h)
return h end},{Value=function(g)for h,i in pairs(f)do pcall(i,g)end end})end
function d.Create(e)return function(f)f=f or{}local g=Instance.new(e)for h,i in
pairs(f)do if h=='__dynamicKeys'and type(i)=='table'then for j,k in pairs(i)do
pcall(function()g[j]=k.Value end)k:Connect(function(l)task.defer(pcall,function(
)if f.__contextKeys and f.__contextKeys._general then f.__contextKeys._general()
end g[j]=f.__contextKeys and f.__contextKeys[j]and f.__contextKeys[j]()or l end)
end)end continue end if typeof(i)=='table'and i.__unique then i.Parent=g end
pcall(function()g[h]=i end)end return setmetatable({__unique=true},{__metatable=
g,__index=function(h,i)if i=='__instance'then return g end local j=g[i]if
typeof(j)=='function'then return function(k,...)return j(g,...)end else return j
end end,__newindex=function(h,i,j)g[i]=j end})end end return d end function a.d(
)local c=a.cache.d if not c then c={c=b()}a.cache.d=c end return c.c end end do
local b=function()local b=a.a()local c=function(c)return b.Clone(game:
GetService(c))end return{HttpService=(c'HttpService'),TweenService=(c
'TweenService'),RunService=(c'RunService'),UserInputService=(c'UserInputService'
),GuiService=(c'GuiService'),Workspace=(c'Workspace'),Players=(c'Players'),
Lighting=(c'Lighting')}end function a.e()local c=a.cache.e if not c then c={c=b(
)}a.cache.e=c end return c.c end end do local b=function()local b=a.e()local c,d
=b.HttpService,{}d.__index=d local e,f,g,h,i={Accent=true,Structures=true,Tabs=
true,Theme=true,__cascadeRecorder=true,__cascadeRecorderNode=true,__container=
true,__window=true,addToHistory=true},{BindPressed=true,Closed=true,Pushed=true,
TextChanged=true,ValueChanged=true},{App={'WindowPill','Theme','Accent'},Window=
{'Title','Subtitle','Searching','Draggable','Resizable','CanExit','CanMinimize',
'CanZoom','Maximized','Minimized','Dropshadow','UIBlur','Size','Position'},
Section={'Title','Disclosure','Expanded','LayoutOrder','Visible'},Tab={'Title',
'Icon','Indentation','Selected','LayoutOrder','Visible'},Page={'LayoutOrder',
'Visible'},PageSection={'Title','Subtitle','LayoutOrder','Visible'},Form={
'LayoutOrder','Visible'},Row={'SearchIndex','LayoutOrder','Visible'},HStack={
'Padding','HorizontalAlignment','VerticalAlignment','LayoutOrder','Visible'},
VStack={'Padding','HorizontalAlignment','VerticalAlignment','LayoutOrder',
'Visible'},TitleStack={'Title','Subtitle','LayoutOrder','Visible'},Label={'Text'
,'RichText','TextXAlignment','TextYAlignment','TextWrapped','LayoutOrder',
'Visible','Size','Position'},Symbol={'Image','Style','LayoutOrder','Visible',
'Size','Position'},ImageSurface={'Image','ImageColor','SurfaceColor','Gradient',
'LayoutOrder','Visible'},Toggle={'Value','LayoutOrder','Visible'},TextField={
'Placeholder','Value','LayoutOrder','Visible'},KeybindField={'Placeholder',
'Value','LayoutOrder','Visible'},Slider={'Minimum','Maximum','Value',
'LayoutOrder','Visible'},Button={'State','Label','LayoutOrder','Visible'},
Stepper={'Minimum','Maximum','Step','Fielded','Value','LayoutOrder','Visible'},
RadioButtonGroup={'Options','Value','LayoutOrder','Visible'},PopUpButton={
'Options','Expanded','Anchor','Maximum','Value','LayoutOrder','Visible'},
PullDownButton={'Options','Expanded','Anchor','Label','Value','LayoutOrder',
'Visible'},Notification={'Title','Subtitle','App','AppIcon','Icon','Duration',
'LayoutOrder','Visible'}},{'Visible','Active','AnchorPoint','Position','Size',
'AutomaticSize','BackgroundColor3','BackgroundTransparency','BorderColor3',
'BorderMode','BorderSizePixel','ClipsDescendants','LayoutOrder','Rotation',
'Selectable','SelectionGroup','Text','RichText','TextColor3','TextTransparency',
'TextSize','TextTruncate','TextWrapped','TextXAlignment','TextYAlignment',
'FontFace','LineHeight','PlaceholderText','Image','ImageColor3',
'ImageTransparency','ImageRectOffset','ImageRectSize','ScaleType','SliceCenter',
'SliceScale','CanvasSize','AutomaticCanvasSize','ScrollBarImageColor3',
'ScrollBarImageTransparency','ScrollBarThickness','Padding','PaddingBottom',
'PaddingLeft','PaddingRight','PaddingTop','CornerRadius','FillDirection',
'HorizontalAlignment','VerticalAlignment','SortOrder','Wraps','Color',
'Transparency','Thickness','Enabled','ResetOnSpawn','DisplayOrder',
'IgnoreGuiInset','ZIndexBehavior'},function(e,f)local g,h=pcall(function()return
e[f]end)if g then return h end return nil end local j,k,l=function(j)if typeof(j
)=='Instance'then return j end if typeof(j)=='table'then local k=i(j,
'__instance')if typeof(k)=='Instance'then return k end end return nil end,
function(j)local k,l=pcall(function()return j:GetFullName()end)return k and l or
j.Name end,function(j)return math.clamp(math.floor(j*255+0.5),0,255)end
local function m(n,o,p)p=p or 0 if p>8 then return{_type='MaxDepth'}end local q=
typeof(n)if q=='nil'or q=='string'or q=='number'or q=='boolean'then return n
elseif q=='function'then return nil elseif q=='Color3'then local r,s,t=l(n.R),l(
n.G),l(n.B)return{_type='Color3',r=r,g=s,b=t,hex=string.format('#%02X%02X%02X',r
,s,t)}elseif q=='UDim'then return{_type='UDim',scale=n.Scale,offset=n.Offset}
elseif q=='UDim2'then return{_type='UDim2',x={scale=n.X.Scale,offset=n.X.Offset}
,y={scale=n.Y.Scale,offset=n.Y.Offset}}elseif q=='Vector2'then return{_type=
'Vector2',x=n.X,y=n.Y}elseif q=='Vector3'then return{_type='Vector3',x=n.X,y=n.Y
,z=n.Z}elseif q=='Rect'then return{_type='Rect',min=m(n.Min,o,p+1),max=m(n.Max,o
,p+1)}elseif q=='NumberRange'then return{_type='NumberRange',min=n.Min,max=n.Max
}elseif q=='NumberSequence'then local r={}for s,t in ipairs(n.Keypoints)do table
.insert(r,{time=t.Time,value=t.Value,envelope=t.Envelope})end return{_type=
'NumberSequence',keypoints=r}elseif q=='ColorSequence'then local r={}for s,t in
ipairs(n.Keypoints)do table.insert(r,{time=t.Time,value=m(t.Value,o,p+1)})end
return{_type='ColorSequence',keypoints=r}elseif q=='EnumItem'then return{_type=
'EnumItem',enum=tostring(n.EnumType),name=n.Name,value=n.Value,path=tostring(n)}
elseif q=='Font'then return{_type='Font',family=n.Family,weight=n.Weight and n.
Weight.Name,style=n.Style and n.Style.Name}elseif q=='Instance'then return{_type
='Instance',className=n.ClassName,name=n.Name,path=k(n)}elseif q~='table'then
return{_type=q,value=tostring(n)}end if o[n]then return{_type=
'CircularReference'}end o[n]=true if n.Value~=nil and typeof(n.Connect)==
'function'then local r={_type='ValueState',value=m(n.Value,o,p+1)}o[n]=nil
return r end local r,s,t=0,0,true for u in pairs(n)do if typeof(u)=='number'and
u%1==0 and u>=1 then s=math.max(s,u)else t=false end r+=1 end local u={}if t and
r==s then for v=1,s do local w=m(n[v],o,p+1)u[v]=w~=nil and w or{_type=
'Unsupported'}end else for v,w in pairs(n)do local x=m(w,o,p+1)if x~=nil then u[
tostring(v)]=x end end end o[n]=nil return u end local n=function(n,o)if e[n]or
f[n]then return false end if typeof(n)~='string'and typeof(n)~='number'then
return false end if typeof(o)=='function'then return false end return true end
local o,p,q=function(o,p,q)local r,s={},{}if q=='App'and p then local t,u=i(p,
'Theme'),i(p,'Accent')if typeof(t)=='table'and t._id then r.Theme=t._id end if
typeof(u)=='table'and u._id then r.Accent=u._id end end if typeof(o)=='table'
then for t,u in pairs(o)do if n(t,u)then local v=m(u,s,0)if v~=nil then r[
tostring(t)]=v end end end end for t,u in ipairs(g[q]or{})do if n(u,true)then
local v=p and i(p,u)local w=m(v,s,0)if w~=nil then r[u]=w end end end return r
end,function(o)local p,q=pcall(function()return o:GetAttributes()end)if not p
then return{}end return m(q,{},0)end,function(o)local p={}for q,r in ipairs(h)do
local s=i(o,r)local t=m(s,{},0)if t~=nil then p[r]=t end end return p end
local function r(s,t)if not s then return nil end local u={className=s.ClassName
,name=s.Name,path=k(s),properties=q(s),attributes=p(s)}if t then u.children={}
for v,w in ipairs(s:GetChildren())do table.insert(u.children,r(w,true))end end
return u end local function s(t,u)local v,w={type=t.Type,properties=o(t.
Properties,t.Object,t.Type),children={}},t.Instance or j(t.Object)if u.
IncludeInstanceProperties~=false and w then v.instance=r(w,false)end for x,y in
ipairs(t.Children)do table.insert(v.children,s(y,u))end return v end local t,u=
function(t)return c:JSONEncode(t)end,function(t)return{Type='App',Properties=nil
,Object=t,Instance=j(t),Children={}}end function d.new(v,w)local x=setmetatable(
{Active=false,App=v,Options=w or{},Root=u(v),_objectNodes=setmetatable({},{
__mode='k'}),_stack={}},d)x._objectNodes[v]=x.Root if typeof(v)=='table'then v.
__cascadeRecorder=x v.__cascadeRecorderNode=x.Root end return x end function d.
FromContext(v)if typeof(v)~='table'then return nil end local w=i(v,
'__cascadeRecorder')if typeof(w)=='table'and getmetatable(w)==d then return w
end return nil end function d:Start(v)if v and v.Reset then self:Clear()end self
.Active=true if typeof(self.App)=='table'then self.App.__cascadeRecorder=self
self.App.__cascadeRecorderNode=self.Root end return self end function d:Stop()
self.Active=false return self end function d:Clear()table.clear(self.Root.
Children)table.clear(self._stack)self._objectNodes=setmetatable({},{__mode='k'})
self._objectNodes[self.App]=self.Root return self end function d:_Begin(v,w,x)if
not self.Active then return nil end local y=self._stack[#self._stack]if not y
then y=i(v,'__cascadeRecorderNode')or self._objectNodes[v]or self.Root end local
z={Type=w,Properties=x,Object=nil,Instance=nil,Children={}}table.insert(y.
Children,z)table.insert(self._stack,z)return z end function d:_Commit(v,w,x)if
not v then return end if self._stack[#self._stack]==v then table.remove(self.
_stack)end v.Object=w v.Instance=x or j(w)if typeof(w)=='table'then self.
_objectNodes[w]=v w.__cascadeRecorder=self w.__cascadeRecorderNode=v end end
function d:_Cancel(v)if self._stack[#self._stack]==v then table.remove(self.
_stack)end end function d:Dump(v)v=v or self.Options or{}local w={format=
'CascadeAppDump',version=1,mode='recorder',root=s(self.Root,v)}if v.
IncludeInstanceTree then w.instanceTree=r(j(self.App),true)end return t(w)end
function d.DumpApp(v,w)local x=d.FromContext(v)if x then return x:Dump(w)end
return t{format='CascadeAppDump',version=1,mode='instance',root=r(j(v),true)}end
return d end function a.f()local c=a.cache.f if not c then c={c=b()}a.cache.f=c
end return c.c end end do local b=function()return{['00Circle']=
'rbxassetid://83028270380365',['00CircleFill']='rbxassetid://86809068648199',[
'00Square']='rbxassetid://122749670129495',['00SquareFill']=
'rbxassetid://113994495859936',['01Circle']='rbxassetid://97226917439387',[
'01CircleFill']='rbxassetid://84587253978749',['01Square']=
'rbxassetid://107219376369036',['01SquareFill']='rbxassetid://107829808355635',[
'02Circle']='rbxassetid://84345480243560',['02CircleFill']=
'rbxassetid://138068434560383',['02Square']='rbxassetid://108430049934958',[
'02SquareFill']='rbxassetid://85661875663721',['03Circle']=
'rbxassetid://98956711188823',['03CircleFill']='rbxassetid://101131309093130',[
'03Square']='rbxassetid://110182185893835',['03SquareFill']=
'rbxassetid://82242680200461',['04Circle']='rbxassetid://132715041479836',[
'04CircleFill']='rbxassetid://113282380257070',['04Square']=
'rbxassetid://70514965693841',['04SquareFill']='rbxassetid://89262337216532',[
'05Circle']='rbxassetid://125294166213948',['05CircleFill']=
'rbxassetid://130265371531723',['05Square']='rbxassetid://107516239217699',[
'05SquareFill']='rbxassetid://117323864732404',['06Circle']=
'rbxassetid://137306965556122',['06CircleFill']='rbxassetid://94892211121096',[
'06Square']='rbxassetid://104889857379096',['06SquareFill']=
'rbxassetid://105314805711265',['07Circle']='rbxassetid://99787513377232',[
'07CircleFill']='rbxassetid://132058802818154',['07Square']=
'rbxassetid://82223513821897',['07SquareFill']='rbxassetid://137613755884312',[
'08Circle']='rbxassetid://88618425851192',['08CircleFill']=
'rbxassetid://105339976565490',['08Square']='rbxassetid://129990117272036',[
'08SquareFill']='rbxassetid://125650076611446',['09Circle']=
'rbxassetid://103471787905743',['09CircleFill']='rbxassetid://105841318829878',[
'09Square']='rbxassetid://110493201929378',['09SquareFill']=
'rbxassetid://107198536694769',['0Circle']='rbxassetid://137352523558390',[
'0CircleFill']='rbxassetid://92718288126091',['0Square']=
'rbxassetid://136892616930971',['0SquareFill']='rbxassetid://71376891799551',[
'10ArrowTriangleheadClockwise']='rbxassetid://128762303542944',[
'10ArrowTriangleheadCounterclockwise']='rbxassetid://76674527536932',[
'10Calendar']='rbxassetid://99971987072382',['10Circle']=
'rbxassetid://92515002741648',['10CircleFill']='rbxassetid://114340053449257',[
'10Lane']='rbxassetid://118673566178183',['10Square']=
'rbxassetid://70604007286548',['10SquareFill']='rbxassetid://110206837615888',[
'11Calendar']='rbxassetid://84425977875050',['11Circle']=
'rbxassetid://98647733126057',['11CircleFill']='rbxassetid://114595490819205',[
'11Lane']='rbxassetid://134292941036759',['11Square']=
'rbxassetid://95453375539358',['11SquareFill']='rbxassetid://132202039821354',[
'12Calendar']='rbxassetid://114493786022821',['12Circle']=
'rbxassetid://121672832794983',['12CircleFill']='rbxassetid://122946923969916',[
'12Lane']='rbxassetid://132443115464407',['12Square']=
'rbxassetid://135913931652877',['12SquareFill']='rbxassetid://70425571060983',[
'13Calendar']='rbxassetid://125476355270691',['13Circle']=
'rbxassetid://124979455738373',['13CircleFill']='rbxassetid://76676158822711',[
'13Square']='rbxassetid://92207708556496',['13SquareFill']=
'rbxassetid://123971106532801',['14Calendar']='rbxassetid://79389761405548',[
'14Circle']='rbxassetid://116170541336582',['14CircleFill']=
'rbxassetid://135867491119117',['14Square']='rbxassetid://136705500158279',[
'14SquareFill']='rbxassetid://114499637922730',['15ArrowTriangleheadClockwise']=
'rbxassetid://110732229815610',['15ArrowTriangleheadCounterclockwise']=
'rbxassetid://127299415437175',['15Calendar']='rbxassetid://81636478363595',[
'15Circle']='rbxassetid://78927448681525',['15CircleFill']=
'rbxassetid://118327895426422',['15Square']='rbxassetid://90508797482214',[
'15SquareFill']='rbxassetid://120440643138154',['16Calendar']=
'rbxassetid://126220874727895',['16Circle']='rbxassetid://126567549131052',[
'16CircleFill']='rbxassetid://126688524438218',['16Square']=
'rbxassetid://84244666904565',['16SquareFill']='rbxassetid://105088240557538',[
'17Calendar']='rbxassetid://77297968626397',['17Circle']=
'rbxassetid://96369047073930',['17CircleFill']='rbxassetid://80691999562724',[
'17Square']='rbxassetid://78926693972141',['17SquareFill']=
'rbxassetid://85270565257338',['18Calendar']='rbxassetid://125707457639914',[
'18Circle']='rbxassetid://103499330032468',['18CircleFill']=
'rbxassetid://139400390011207',['18Square']='rbxassetid://96425450360158',[
'18SquareFill']='rbxassetid://95747860980425',['19Calendar']=
'rbxassetid://132759347346313',['19Circle']='rbxassetid://136880006821425',[
'19CircleFill']='rbxassetid://132010253426021',['19Square']=
'rbxassetid://86547351573625',['19SquareFill']='rbxassetid://113507822362885',[
'1Brakesignal']='rbxassetid://102491003904804',['1Calendar']=
'rbxassetid://98185546831379',['1Circle']='rbxassetid://130621158480059',[
'1CircleFill']='rbxassetid://119914746762070',['1Lane']=
'rbxassetid://108289649639933',['1Magnifyingglass']=
'rbxassetid://79094731969211',['1Square']='rbxassetid://89938443056690',[
'1SquareFill']='rbxassetid://91936609218444',['20Calendar']=
'rbxassetid://128748725884252',['20Circle']='rbxassetid://74724597303762',[
'20CircleFill']='rbxassetid://74025988368489',['20Square']=
'rbxassetid://100301580149753',['20SquareFill']='rbxassetid://81847976697662',[
'21Calendar']='rbxassetid://92479265735150',['21Circle']=
'rbxassetid://107042620932339',['21CircleFill']='rbxassetid://86466285996936',[
'21Square']='rbxassetid://118914310058881',['21SquareFill']=
'rbxassetid://137245696391587',['22Calendar']='rbxassetid://76926833805322',[
'22Circle']='rbxassetid://112949971675201',['22CircleFill']=
'rbxassetid://136675480787222',['22Square']='rbxassetid://70382544382768',[
'22SquareFill']='rbxassetid://136349772275086',['23Calendar']=
'rbxassetid://72540972332711',['23Circle']='rbxassetid://122644356502405',[
'23CircleFill']='rbxassetid://132927065976368',['23Square']=
'rbxassetid://133241790954057',['23SquareFill']='rbxassetid://131940949142587',[
'24Calendar']='rbxassetid://108261967636040',['24Circle']=
'rbxassetid://82428286993981',['24CircleFill']='rbxassetid://91411670826506',[
'24Square']='rbxassetid://139919867683269',['24SquareFill']=
'rbxassetid://71799607296423',['25Calendar']='rbxassetid://100633222933029',[
'25Circle']='rbxassetid://100704542539329',['25CircleFill']=
'rbxassetid://86282617997408',['25Square']='rbxassetid://85711261321247',[
'25SquareFill']='rbxassetid://92036217304787',['26Calendar']=
'rbxassetid://88912330770844',['26Circle']='rbxassetid://115236591138732',[
'26CircleFill']='rbxassetid://83556043270110',['26Square']=
'rbxassetid://138273926712016',['26SquareFill']='rbxassetid://104321077565351',[
'27Calendar']='rbxassetid://73201335235154',['27Circle']=
'rbxassetid://121764559253612',['27CircleFill']='rbxassetid://100219016046799',[
'27Square']='rbxassetid://83717779381903',['27SquareFill']=
'rbxassetid://99650567921303',['28Calendar']='rbxassetid://105788634273895',[
'28Circle']='rbxassetid://109395541199407',['28CircleFill']=
'rbxassetid://129403579574600',['28Square']='rbxassetid://101847553369506',[
'28SquareFill']='rbxassetid://125178103545775',['29Calendar']=
'rbxassetid://128726400170465',['29Circle']='rbxassetid://87253975016702',[
'29CircleFill']='rbxassetid://121458185459649',['29Square']=
'rbxassetid://115063859229340',['29SquareFill']='rbxassetid://86136990319710',[
'2Brakesignal']='rbxassetid://88382719031732',['2Calendar']=
'rbxassetid://98050312641153',['2Circle']='rbxassetid://88686989179529',[
'2CircleFill']='rbxassetid://117347647638182',['2Lane']=
'rbxassetid://73043736430431',['2Square']='rbxassetid://81405613158962',[
'2SquareFill']='rbxassetid://135108412068288',['2h']=
'rbxassetid://120554166840614',['2hCircle']='rbxassetid://80587972141848',[
'2hCircleFill']='rbxassetid://121984358372075',['30ArrowTriangleheadClockwise']=
'rbxassetid://74624510136124',['30ArrowTriangleheadCounterclockwise']=
'rbxassetid://135632602776767',['30Calendar']='rbxassetid://77714024284758',[
'30Circle']='rbxassetid://125851565099112',['30CircleFill']=
'rbxassetid://134873563219388',['30Square']='rbxassetid://98178040903351',[
'30SquareFill']='rbxassetid://124000446507321',['31Calendar']=
'rbxassetid://103712950051496',['31Circle']='rbxassetid://82832572967769',[
'31CircleFill']='rbxassetid://84776901062406',['31Square']=
'rbxassetid://100093966297675',['31SquareFill']='rbxassetid://109439143484113',[
'32Circle']='rbxassetid://128828563808013',['32CircleFill']=
'rbxassetid://106794836279809',['32Square']='rbxassetid://100529952897522',[
'32SquareFill']='rbxassetid://135053972405086',['33Circle']=
'rbxassetid://76834446462070',['33CircleFill']='rbxassetid://133471374143015',[
'33Square']='rbxassetid://132109947433846',['33SquareFill']=
'rbxassetid://92211719099524',['34Circle']='rbxassetid://78797665476071',[
'34CircleFill']='rbxassetid://87668137780853',['34Square']=
'rbxassetid://81598346353759',['34SquareFill']='rbxassetid://111975400463928',[
'35Circle']='rbxassetid://72637078655736',['35CircleFill']=
'rbxassetid://76736961743352',['35Square']='rbxassetid://119727141136934',[
'35SquareFill']='rbxassetid://117373278288844',['36Circle']=
'rbxassetid://98319312866253',['36CircleFill']='rbxassetid://77316378982918',[
'36Square']='rbxassetid://101616504680112',['36SquareFill']=
'rbxassetid://96022911959763',['37Circle']='rbxassetid://93687894376513',[
'37CircleFill']='rbxassetid://86485501852806',['37Square']=
'rbxassetid://139267911387901',['37SquareFill']='rbxassetid://91193382294091',[
'38Circle']='rbxassetid://121521469341757',['38CircleFill']=
'rbxassetid://105792148160711',['38Square']='rbxassetid://106051044850392',[
'38SquareFill']='rbxassetid://138218593875239',['39Circle']=
'rbxassetid://125288441586843',['39CircleFill']='rbxassetid://115892557058939',[
'39Square']='rbxassetid://112598329033121',['39SquareFill']=
'rbxassetid://98638503271042',['3Calendar']='rbxassetid://90795649820662',[
'3Circle']='rbxassetid://85824504644997',['3CircleFill']=
'rbxassetid://74910224380095',['3Lane']='rbxassetid://75686915457526',['3Square'
]='rbxassetid://120223972887350',['3SquareFill']='rbxassetid://84686638879122',[
'40Circle']='rbxassetid://83180478546770',['40CircleFill']=
'rbxassetid://132728837830874',['40Square']='rbxassetid://112224567454433',[
'40SquareFill']='rbxassetid://101941487357695',['41Circle']=
'rbxassetid://75225134548534',['41CircleFill']='rbxassetid://136094650593344',[
'41Square']='rbxassetid://125817109698445',['41SquareFill']=
'rbxassetid://138876358254873',['42Circle']='rbxassetid://127374826103215',[
'42CircleFill']='rbxassetid://122701075418215',['42Square']=
'rbxassetid://138197946562353',['42SquareFill']='rbxassetid://107430077421870',[
'43Circle']='rbxassetid://104838885962058',['43CircleFill']=
'rbxassetid://133602186789762',['43Square']='rbxassetid://108869662941519',[
'43SquareFill']='rbxassetid://101067151905278',['44Circle']=
'rbxassetid://128426936704844',['44CircleFill']='rbxassetid://137168247083835',[
'44Square']='rbxassetid://75545715410292',['44SquareFill']=
'rbxassetid://111628503136844',['45ArrowTriangleheadClockwise']=
'rbxassetid://120134525176078',['45ArrowTriangleheadCounterclockwise']=
'rbxassetid://123087333385500',['45Circle']='rbxassetid://102436156063197',[
'45CircleFill']='rbxassetid://132306924075130',['45Square']=
'rbxassetid://125303441223733',['45SquareFill']='rbxassetid://80893347771788',[
'46Circle']='rbxassetid://118709156698649',['46CircleFill']=
'rbxassetid://88412402128165',['46Square']='rbxassetid://101875634976749',[
'46SquareFill']='rbxassetid://123057980319615',['47Circle']=
'rbxassetid://134558301013886',['47CircleFill']='rbxassetid://137987636103436',[
'47Square']='rbxassetid://99547664530529',['47SquareFill']=
'rbxassetid://82416495107377',['48Circle']='rbxassetid://116659923987327',[
'48CircleFill']='rbxassetid://127585613163244',['48Square']=
'rbxassetid://88269026511583',['48SquareFill']='rbxassetid://128550656233484',[
'49Circle']='rbxassetid://116042725943528',['49CircleFill']=
'rbxassetid://70466830951089',['49Square']='rbxassetid://108294144287133',[
'49SquareFill']='rbxassetid://136824579664757',['4AltCircle']=
'rbxassetid://96065443995675',['4AltCircleFill']='rbxassetid://114108784117364',
['4AltSquare']='rbxassetid://92007544707653',['4AltSquareFill']=
'rbxassetid://76782617943156',['4Calendar']='rbxassetid://117683067057146',[
'4Circle']='rbxassetid://85410335884890',['4CircleFill']=
'rbxassetid://117232689412009',['4Lane']='rbxassetid://90837743623016',[
'4Square']='rbxassetid://93614529304671',['4SquareFill']=
'rbxassetid://100548465120334',['4a']='rbxassetid://88536395160758',['4aCircle']
='rbxassetid://121454839355516',['4aCircleFill']='rbxassetid://107149222480560',
['4h']='rbxassetid://90250390247173',['4hCircle']='rbxassetid://128648031725106'
,['4hCircleFill']='rbxassetid://107728946853252',['4kTv']=
'rbxassetid://123674016506451',['4kTvFill']='rbxassetid://75999084684547',['4l']
='rbxassetid://106811124514270',['4lCircle']='rbxassetid://135697353497465',[
'4lCircleFill']='rbxassetid://80779362494066',['50Circle']=
'rbxassetid://135049337228431',['50CircleFill']='rbxassetid://125135742443046',[
'50Square']='rbxassetid://92307459965921',['50SquareFill']=
'rbxassetid://129462028494317',['5ArrowTriangleheadClockwise']=
'rbxassetid://107383111794341',['5ArrowTriangleheadCounterclockwise']=
'rbxassetid://135478288864818',['5Calendar']='rbxassetid://139653164527127',[
'5Circle']='rbxassetid://138643256382369',['5CircleFill']=
'rbxassetid://81772533269342',['5Lane']='rbxassetid://110377621267035',[
'5Square']='rbxassetid://91863717310443',['5SquareFill']=
'rbxassetid://101252363013967',['60ArrowTriangleheadClockwise']=
'rbxassetid://99108772093793',['60ArrowTriangleheadCounterclockwise']=
'rbxassetid://139258134359335',['6AltCircle']='rbxassetid://120350071573739',[
'6AltCircleFill']='rbxassetid://86316701823078',['6AltSquare']=
'rbxassetid://137257337817731',['6AltSquareFill']='rbxassetid://80160043116436',
['6Calendar']='rbxassetid://90102440243055',['6Circle']=
'rbxassetid://72149384954065',['6CircleFill']='rbxassetid://78297067061715',[
'6Lane']='rbxassetid://100354087243931',['6Square']=
'rbxassetid://120439558590340',['6SquareFill']='rbxassetid://72841586020208',[
'75ArrowTriangleheadClockwise']='rbxassetid://89560078813691',[
'75ArrowTriangleheadCounterclockwise']='rbxassetid://72011197335579',[
'7Calendar']='rbxassetid://86716901913977',['7Circle']=
'rbxassetid://134362216731609',['7CircleFill']='rbxassetid://128766124434672',[
'7Lane']='rbxassetid://87939210980451',['7Square']='rbxassetid://83019303763050'
,['7SquareFill']='rbxassetid://73394265174480',['8Calendar']=
'rbxassetid://123022135128239',['8Circle']='rbxassetid://121463881556470',[
'8CircleFill']='rbxassetid://98677569612860',['8Lane']=
'rbxassetid://73530448957022',['8Square']='rbxassetid://94534023907446',[
'8SquareFill']='rbxassetid://85132457001862',['90ArrowTriangleheadClockwise']=
'rbxassetid://75428859613511',['90ArrowTriangleheadCounterclockwise']=
'rbxassetid://100354802037825',['9AltCircle']='rbxassetid://116943330734703',[
'9AltCircleFill']='rbxassetid://74872216618128',['9AltSquare']=
'rbxassetid://93297089978709',['9AltSquareFill']='rbxassetid://118501519632418',
['9Calendar']='rbxassetid://134402469311043',['9Circle']=
'rbxassetid://125766582985411',['9CircleFill']='rbxassetid://140040901574476',[
'9Lane']='rbxassetid://114357239781363',['9Square']=
'rbxassetid://99344731424187',['9SquareFill']='rbxassetid://82741160180392',
aCircle='rbxassetid://91422740186540',aCircleFill='rbxassetid://80176355279906',
aSquare='rbxassetid://110661926393175',aSquareFill=
'rbxassetid://118925074578922',abs='rbxassetid://104303148312310',absBrakesignal
='rbxassetid://108288270804521',absBrakesignalSlash=
'rbxassetid://139900347181805',absCircle='rbxassetid://127912426662073',
absCircleFill='rbxassetid://90389741944345',ac='rbxassetid://70584542126579',
acSlash='rbxassetid://131379895015246',accessibility=
'rbxassetid://117778752008722',accessibilityBadgeArrowUpRight=
'rbxassetid://88644725338334',accessibilityFill='rbxassetid://99814938584597',
airCarSide='rbxassetid://98549379001911',airCarSideFill=
'rbxassetid://74763317863435',airConditionerHorizontal=
'rbxassetid://110142871235385',airConditionerHorizontalFill=
'rbxassetid://103206558750136',airConditionerVertical=
'rbxassetid://140704008468110',airConditionerVerticalFill=
'rbxassetid://127386997506238',airConvertibleSide='rbxassetid://94382667709805',
airConvertibleSideFill='rbxassetid://82469124247932',airPickupSide=
'rbxassetid://118486435215683',airPickupSideFill='rbxassetid://117535325295275',
airPurifier='rbxassetid://121219934750735',airPurifierFill=
'rbxassetid://121283023382392',airSuvSide='rbxassetid://85264461833523',
airSuvSideFill='rbxassetid://97290218602252',airplane=
'rbxassetid://133302594398057',airplaneArrival='rbxassetid://83294282884497',
airplaneCircle='rbxassetid://80486171410882',airplaneCircleFill=
'rbxassetid://128766972660071',airplaneCloud='rbxassetid://106395273712591',
airplaneDeparture='rbxassetid://110438799431401',airplaneLanded=
'rbxassetid://73211840820224',airplanePathDotted='rbxassetid://77257428977392',
airplaneTicket='rbxassetid://131221608879931',airplaneTicketFill=
'rbxassetid://86885394248735',airplaneUpForward='rbxassetid://107258551084271',
airplaneUpForwardApp='rbxassetid://79498952458569',airplaneUpForwardAppFill=
'rbxassetid://107908472496037',airplaneUpRight='rbxassetid://106459716120779',
airplaneUpRightApp='rbxassetid://110247582997305',airplaneUpRightAppFill=
'rbxassetid://101889444884144',airplaneseat='rbxassetid://127474636630226',
airplayAudio='rbxassetid://97872416468292',airplayAudioBadgeExclamationmark=
'rbxassetid://82691022378697',airplayAudioCircle='rbxassetid://79355534338385',
airplayAudioCircleFill='rbxassetid://99882830125971',airplayVideo=
'rbxassetid://116428826763896',airplayVideoBadgeExclamationmark=
'rbxassetid://109306301769770',airplayVideoCircle='rbxassetid://124272728652470'
,airplayVideoCircleFill='rbxassetid://85432097318270',airpodGen3Left=
'rbxassetid://82259989716387',airpodGen3Right='rbxassetid://137315291748188',
airpodLeft='rbxassetid://111640685320561',airpodRight=
'rbxassetid://92417752017903',airpods='rbxassetid://102934461761467',
airpodsChargingcase='rbxassetid://137188300011249',airpodsChargingcaseFill=
'rbxassetid://121114519994300',airpodsChargingcaseWireless=
'rbxassetid://82987264570808',airpodsChargingcaseWirelessFill=
'rbxassetid://115642815762700',airpodsGen3='rbxassetid://77127013352192',
airpodsGen3ChargingcaseWireless='rbxassetid://103252568691488',
airpodsGen3ChargingcaseWirelessFill='rbxassetid://138720486088620',airpodsGen4=
'rbxassetid://98813390596396',airpodsGen4ChargingcaseWireless=
'rbxassetid://81680510405013',airpodsGen4ChargingcaseWirelessFill=
'rbxassetid://103031303446000',airpodsGen4Left='rbxassetid://80342628187549',
airpodsGen4Right='rbxassetid://116899672566620',airpodsMax=
'rbxassetid://129315211901723',airpodsPro='rbxassetid://115908103846782',
airpodsProChargingcaseWireless='rbxassetid://100805444479847',
airpodsProChargingcaseWirelessFill='rbxassetid://136683531844341',
airpodsProChargingcaseWirelessRadiowavesLeftAndRight=
'rbxassetid://114165652099787',
airpodsProChargingcaseWirelessRadiowavesLeftAndRightFill=
'rbxassetid://101029375177334',airpodsProLeft='rbxassetid://76398589107645',
airpodsProRight='rbxassetid://110293485717436',airportExpress=
'rbxassetid://84914509690369',airportExtreme='rbxassetid://91752831773330',
airportExtremeTower='rbxassetid://84855334762457',airtag=
'rbxassetid://133510297942223',airtagFill='rbxassetid://93526399862084',
airtagRadiowavesForward='rbxassetid://70835353797393',
airtagRadiowavesForwardFill='rbxassetid://86335759121702',alarm=
'rbxassetid://94592263823729',alarmFill='rbxassetid://110854125435299',
alarmWavesLeftAndRight='rbxassetid://102548399209045',alarmWavesLeftAndRightFill
='rbxassetid://134148018537010',alignHorizontalCenter=
'rbxassetid://105387534506369',alignHorizontalCenterFill=
'rbxassetid://80228868536795',alignHorizontalLeft='rbxassetid://121417857557559'
,alignHorizontalLeftFill='rbxassetid://100445284088857',alignHorizontalRight=
'rbxassetid://78727045612818',alignHorizontalRightFill=
'rbxassetid://117183604846016',alignVerticalBottom=
'rbxassetid://126811262141729',alignVerticalBottomFill=
'rbxassetid://82585565029168',alignVerticalCenter='rbxassetid://87497476785272',
alignVerticalCenterFill='rbxassetid://117303443763648',alignVerticalTop=
'rbxassetid://131829541174867',alignVerticalTopFill=
'rbxassetid://101695426589183',allergens='rbxassetid://121178963871633',
allergensFill='rbxassetid://120835453624168',alt='rbxassetid://109200502181897',
alternatingcurrent='rbxassetid://139782924101096',americanFootball=
'rbxassetid://131900707633129',americanFootballCircle=
'rbxassetid://126285687669632',americanFootballCircleFill=
'rbxassetid://120792391598468',americanFootballFill=
'rbxassetid://79772090879526',americanFootballProfessional=
'rbxassetid://79730291560074',americanFootballProfessionalCircle=
'rbxassetid://81165175257890',americanFootballProfessionalCircleFill=
'rbxassetid://85819536267258',americanFootballProfessionalFill=
'rbxassetid://132612978101644',amplifier='rbxassetid://99850964394213',angle=
'rbxassetid://97688144611384',ant='rbxassetid://108437719613458',antCircle=
'rbxassetid://73241676267354',antCircleFill='rbxassetid://90861066249394',
antFill='rbxassetid://96049362790915',antennaRadiowavesLeftAndRight=
'rbxassetid://105783651610996',antennaRadiowavesLeftAndRightCircle=
'rbxassetid://106918794715121',antennaRadiowavesLeftAndRightCircleFill=
'rbxassetid://95673268616759',antennaRadiowavesLeftAndRightSlash=
'rbxassetid://83746876033500',antennaRadiowavesLeftAndRightSlashCircle=
'rbxassetid://72562971579727',antennaRadiowavesLeftAndRightSlashCircleFill=
'rbxassetid://80891332949617',app='rbxassetid://94635193120828',
appBackgroundDotted='rbxassetid://109494182446738',appBadge=
'rbxassetid://139613945369115',appBadgeCheckmark='rbxassetid://81426112675395',
appBadgeCheckmarkFill='rbxassetid://93503980447430',appBadgeClock=
'rbxassetid://134458400266753',appBadgeClockFill='rbxassetid://91313378758113',
appBadgeFill='rbxassetid://88296846458468',appConnectedToAppBelowFill=
'rbxassetid://100342957440925',appDashed='rbxassetid://122696783855791',appFill=
'rbxassetid://134138664054034',appGift='rbxassetid://88383425466655',appGiftFill
='rbxassetid://88482932848630',appGrid='rbxassetid://112293112122644',appShadow=
'rbxassetid://74201552367475',appSpecular='rbxassetid://88620890438453',
appTranslucent='rbxassetid://93640553211023',appclip=
'rbxassetid://76150191896193',appendPage='rbxassetid://132003569675874',
appendPageFill='rbxassetid://114109887686405',appleBooksPages=
'rbxassetid://83129377869855',appleBooksPagesFill='rbxassetid://79375977225236',
appleClassicalPages='rbxassetid://109032335324915',appleClassicalPagesFill=
'rbxassetid://73169137754796',appleHapticsAndExclamationmarkTriangle=
'rbxassetid://84679413093806',appleHapticsAndMusicNote=
'rbxassetid://94992791728394',appleHapticsAndMusicNoteSlash=
'rbxassetid://120030098404409',appleHomekit='rbxassetid://138475901062724',
appleImagePlayground='rbxassetid://109144440237311',appleImagePlaygroundFill=
'rbxassetid://95169627623891',appleIntelligence='rbxassetid://92927940894079',
appleIntelligenceBadgeXmark='rbxassetid://136847756879125',appleLogo=
'rbxassetid://74231414416316',appleMeditate='rbxassetid://110884202603539',
appleMeditateCircle='rbxassetid://96265715307080',appleMeditateCircleFill=
'rbxassetid://112011124705717',appleMeditateSquareStack=
'rbxassetid://94877406532991',appleMeditateSquareStackFill=
'rbxassetid://89004763882356',applePodcastsPages='rbxassetid://81110736056917',
applePodcastsPagesFill='rbxassetid://133340247658961',appleTerminal=
'rbxassetid://88837948823350',appleTerminalCircle='rbxassetid://99041514023806',
appleTerminalCircleFill='rbxassetid://100608272114701',appleTerminalFill=
'rbxassetid://83023410889826',appleTerminalOnRectangle=
'rbxassetid://82511073936391',appleTerminalOnRectangleFill=
'rbxassetid://137027325971487',appleWritingTools='rbxassetid://74586880049933',
applepencil='rbxassetid://90464036055920',applepencilAdapterUsbC=
'rbxassetid://116737446576584',applepencilAdapterUsbCFill=
'rbxassetid://139496379804400',applepencilAndScribble=
'rbxassetid://106121855601154',applepencilDoubletap=
'rbxassetid://135219023747373',applepencilGen1='rbxassetid://110959209071798',
applepencilGen2='rbxassetid://136157715411470',applepencilHover=
'rbxassetid://132499964823193',applepencilSqueeze='rbxassetid://130017747173569'
,applepencilTip='rbxassetid://91593400796177',applescript=
'rbxassetid://123810491451954',applescriptFill='rbxassetid://127519934422013',
appletv='rbxassetid://102962906677124',appletvBadgeCheckmark=
'rbxassetid://101652457992538',appletvBadgeCheckmarkFill=
'rbxassetid://78422425043458',appletvBadgeExclamationmark=
'rbxassetid://81489066245182',appletvBadgeExclamationmarkFill=
'rbxassetid://114636492149768',appletvFill='rbxassetid://135903140150465',
appletvremoteGen1='rbxassetid://135270967631693',appletvremoteGen1Fill=
'rbxassetid://81889369734284',appletvremoteGen2='rbxassetid://113895435393863',
appletvremoteGen2Fill='rbxassetid://70483371792990',appletvremoteGen3=
'rbxassetid://105163273755884',appletvremoteGen3Fill=
'rbxassetid://115418422308741',appletvremoteGen4='rbxassetid://100567117806558',
appletvremoteGen4Fill='rbxassetid://113022815763894',applewatch=
'rbxassetid://93138144347379',applewatchAndArrowForward=
'rbxassetid://110039352060422',applewatchBadgeCheckmark=
'rbxassetid://92778985550706',applewatchBadgeExclamationmark=
'rbxassetid://129599825985183',applewatchCaseSizes=
'rbxassetid://109653548205740',applewatchRadiowavesLeftAndRight=
'rbxassetid://102293766705017',applewatchSideRight='rbxassetid://73482822688773'
,applewatchSlash='rbxassetid://79088068041897',applewatchWatchface=
'rbxassetid://83969866132522',appsIpad='rbxassetid://133433158213093',
appsIpadBadgeCheckmark='rbxassetid://117680941737775',appsIpadBadgePlus=
'rbxassetid://88752826248322',appsIpadLandscape='rbxassetid://86493684817099',
appsIpadOnRectanglePortraitDashed='rbxassetid://114536357023471',appsIphone=
'rbxassetid://129167692551812',appsIphoneBadgeCheckmark=
'rbxassetid://103980567296855',appsIphoneBadgePlus='rbxassetid://71713357676686'
,appsIphoneLandscape='rbxassetid://97618850718449',appwindowSwipeRectangle=
'rbxassetid://86269047484283',aqiHigh='rbxassetid://124283236192520',aqiLow=
'rbxassetid://125965149266356',aqiMedium='rbxassetid://121844299793985',
aqiMediumGaugeOpen='rbxassetid://137218283152183',arcadeStick=
'rbxassetid://136881673658280',arcadeStickAndArrowDown=
'rbxassetid://106511231849553',arcadeStickAndArrowLeft=
'rbxassetid://126374741762139',arcadeStickAndArrowLeftAndArrowRightOutward=
'rbxassetid://138813689219871',arcadeStickAndArrowRight=
'rbxassetid://118159594368933',arcadeStickAndArrowUp=
'rbxassetid://86240487616233',arcadeStickAndArrowUpAndArrowDown=
'rbxassetid://112293103027077',arcadeStickConsole='rbxassetid://100669752883004'
,arcadeStickConsoleFill='rbxassetid://122547243212983',archivebox=
'rbxassetid://121345229961059',archiveboxCircle='rbxassetid://98842142630967',
archiveboxCircleFill='rbxassetid://118356120505684',archiveboxFill=
'rbxassetid://93301170884322',arkit='rbxassetid://137506195249781',
arkitBadgeXmark='rbxassetid://112860435555999',arrow2Squarepath=
'rbxassetid://115537208650468',arrow3Trianglepath='rbxassetid://81605064455205',
arrowBackward='rbxassetid://88649419574770',arrowBackwardCircle=
'rbxassetid://136956454087601',arrowBackwardCircleDotted=
'rbxassetid://102449299281450',arrowBackwardCircleFill=
'rbxassetid://113290948178109',arrowBackwardSquare='rbxassetid://98664430133231'
,arrowBackwardSquareFill='rbxassetid://118804167090498',arrowBackwardToLine=
'rbxassetid://110504526302150',arrowBackwardToLineCircle=
'rbxassetid://78052803011890',arrowBackwardToLineCircleFill=
'rbxassetid://107890859844246',arrowBackwardToLineCompact=
'rbxassetid://138459990868775',arrowBackwardToLineSquare=
'rbxassetid://78531808011930',arrowBackwardToLineSquareFill=
'rbxassetid://75065759600319',arrowClockwise='rbxassetid://135150629431078',
arrowClockwiseCircle='rbxassetid://74157022752988',arrowClockwiseCircleFill=
'rbxassetid://116449962863671',arrowClockwiseSquare=
'rbxassetid://119196146574286',arrowClockwiseSquareFill=
'rbxassetid://134619956093951',arrowCounterclockwise=
'rbxassetid://93269551188316',arrowCounterclockwiseCircle=
'rbxassetid://100324999603105',arrowCounterclockwiseCircleFill=
'rbxassetid://89831785654730',arrowCounterclockwiseSquare=
'rbxassetid://120748383469686',arrowCounterclockwiseSquareFill=
'rbxassetid://100952327039099',arrowDown='rbxassetid://104294868419018',
arrowDownAndLineHorizontalAndArrowUp='rbxassetid://87886576322262',arrowDownApp=
'rbxassetid://126624351763492',arrowDownAppDashed='rbxassetid://120863880942317'
,arrowDownAppDashedTrianglebadgeExclamationmark='rbxassetid://102776085772699',
arrowDownAppFill='rbxassetid://117366485860412',arrowDownApplewatch=
'rbxassetid://90433452282294',arrowDownBackward='rbxassetid://118854524118128',
arrowDownBackwardAndArrowUpForward='rbxassetid://129996365679960',
arrowDownBackwardAndArrowUpForwardCircle='rbxassetid://106065835022812',
arrowDownBackwardAndArrowUpForwardCircleFill='rbxassetid://119097194369411',
arrowDownBackwardAndArrowUpForwardRectangle='rbxassetid://86325793969716',
arrowDownBackwardAndArrowUpForwardRectangleFill='rbxassetid://106014356084217',
arrowDownBackwardAndArrowUpForwardSquare='rbxassetid://95759279304249',
arrowDownBackwardAndArrowUpForwardSquareFill='rbxassetid://101483128565522',
arrowDownBackwardCircle='rbxassetid://101148976029987',
arrowDownBackwardCircleDotted='rbxassetid://101994800462988',
arrowDownBackwardCircleFill='rbxassetid://104416395931960',
arrowDownBackwardSquare='rbxassetid://120139080263601',
arrowDownBackwardSquareFill='rbxassetid://96397200712935',
arrowDownBackwardToptrailingRectangle='rbxassetid://128098505832607',
arrowDownBackwardToptrailingRectangleFill='rbxassetid://136959509151256',
arrowDownCircle='rbxassetid://135587411885714',arrowDownCircleBadgePause=
'rbxassetid://124364310110740',arrowDownCircleBadgePauseFill=
'rbxassetid://126034849419398',arrowDownCircleBadgeXmark=
'rbxassetid://132627732829179',arrowDownCircleBadgeXmarkFill=
'rbxassetid://118370194181839',arrowDownCircleDotted=
'rbxassetid://127920779547879',arrowDownCircleFill=
'rbxassetid://126613187533372',arrowDownDocument='rbxassetid://90055782991587',
arrowDownDocumentFill='rbxassetid://79473920816292',arrowDownForward=
'rbxassetid://139974283617157',arrowDownForwardAndArrowUpBackward=
'rbxassetid://127715462923536',arrowDownForwardAndArrowUpBackwardCircle=
'rbxassetid://99167961428062',arrowDownForwardAndArrowUpBackwardCircleFill=
'rbxassetid://90718624284226',arrowDownForwardAndArrowUpBackwardRectangle=
'rbxassetid://109896428717818',arrowDownForwardAndArrowUpBackwardRectangleFill=
'rbxassetid://101296521578239',arrowDownForwardAndArrowUpBackwardSquare=
'rbxassetid://79718568224638',arrowDownForwardAndArrowUpBackwardSquareFill=
'rbxassetid://118653902474315',arrowDownForwardCircle=
'rbxassetid://91129944003312',arrowDownForwardCircleDotted=
'rbxassetid://79058882401591',arrowDownForwardCircleFill=
'rbxassetid://90616558170424',arrowDownForwardSquare=
'rbxassetid://135965851538670',arrowDownForwardSquareFill=
'rbxassetid://86743623269835',arrowDownForwardTopleadingRectangle=
'rbxassetid://100959335820912',arrowDownForwardTopleadingRectangleFill=
'rbxassetid://116322978531995',arrowDownHeart='rbxassetid://136960602456421',
arrowDownHeartFill='rbxassetid://111091347386558',arrowDownLeft=
'rbxassetid://122224910211071',arrowDownLeftAndArrowUpRight=
'rbxassetid://104164888472588',arrowDownLeftAndArrowUpRightCircle=
'rbxassetid://139958788849042',arrowDownLeftAndArrowUpRightCircleFill=
'rbxassetid://77940274374626',arrowDownLeftAndArrowUpRightRectangle=
'rbxassetid://130268567739782',arrowDownLeftAndArrowUpRightRectangleFill=
'rbxassetid://136447163738546',arrowDownLeftAndArrowUpRightSquare=
'rbxassetid://74329478808503',arrowDownLeftAndArrowUpRightSquareFill=
'rbxassetid://112488682508039',arrowDownLeftArrowUpRight=
'rbxassetid://110501884912653',arrowDownLeftArrowUpRightCircle=
'rbxassetid://81099073033121',arrowDownLeftArrowUpRightCircleFill=
'rbxassetid://101109754050167',arrowDownLeftArrowUpRightSquare=
'rbxassetid://88334237843710',arrowDownLeftArrowUpRightSquareFill=
'rbxassetid://118701870591849',arrowDownLeftCircle='rbxassetid://92547732629924'
,arrowDownLeftCircleDotted='rbxassetid://95151928004771',arrowDownLeftCircleFill
='rbxassetid://86643102687783',arrowDownLeftSquare='rbxassetid://83024806847192'
,arrowDownLeftSquareFill='rbxassetid://112644786722286',
arrowDownLeftToprightRectangle='rbxassetid://113141341051779',
arrowDownLeftToprightRectangleFill='rbxassetid://130388914126671',
arrowDownLeftVideo='rbxassetid://76350513151515',arrowDownLeftVideoFill=
'rbxassetid://140633155175462',arrowDownMessage='rbxassetid://100621717832807',
arrowDownMessageFill='rbxassetid://74100213634491',arrowDownRight=
'rbxassetid://79448572967402',arrowDownRightAndArrowUpLeft=
'rbxassetid://89225125347394',arrowDownRightAndArrowUpLeftCircle=
'rbxassetid://80802938078543',arrowDownRightAndArrowUpLeftCircleFill=
'rbxassetid://140259126181191',arrowDownRightAndArrowUpLeftRectangle=
'rbxassetid://107575531690549',arrowDownRightAndArrowUpLeftRectangleFill=
'rbxassetid://113365716944083',arrowDownRightAndArrowUpLeftSquare=
'rbxassetid://120050363028423',arrowDownRightAndArrowUpLeftSquareFill=
'rbxassetid://131765067257388',arrowDownRightCircle=
'rbxassetid://134197654580585',arrowDownRightCircleDotted=
'rbxassetid://90832353961844',arrowDownRightCircleFill=
'rbxassetid://74812371090023',arrowDownRightSquare=
'rbxassetid://140176759330522',arrowDownRightSquareFill=
'rbxassetid://123563260022187',arrowDownRightTopleftRectangle=
'rbxassetid://71596950769323',arrowDownRightTopleftRectangleFill=
'rbxassetid://93249639234250',arrowDownSquare='rbxassetid://96495851984147',
arrowDownSquareFill='rbxassetid://78941122620807',arrowDownToLine=
'rbxassetid://81945854315577',arrowDownToLineCircle=
'rbxassetid://104062291929801',arrowDownToLineCircleFill=
'rbxassetid://90141383354318',arrowDownToLineCompact=
'rbxassetid://75925803459799',arrowDownToLineSquare=
'rbxassetid://105349654584160',arrowDownToLineSquareFill=
'rbxassetid://79175779364203',arrowForward='rbxassetid://121760727871708',
arrowForwardCircle='rbxassetid://122188146321824',arrowForwardCircleDotted=
'rbxassetid://84632482795162',arrowForwardCircleFill=
'rbxassetid://136358992052631',arrowForwardFolder='rbxassetid://110107371036130'
,arrowForwardFolderFill='rbxassetid://132247910751681',arrowForwardSquare=
'rbxassetid://72679546839781',arrowForwardSquareFill=
'rbxassetid://90595336174638',arrowForwardToLine='rbxassetid://132984755317478',
arrowForwardToLineCircle='rbxassetid://130410583070504',
arrowForwardToLineCircleFill='rbxassetid://88891501791499',
arrowForwardToLineCompact='rbxassetid://96760155810467',arrowForwardToLineSquare
='rbxassetid://97107689023381',arrowForwardToLineSquareFill=
'rbxassetid://140368370217658',arrowLeft='rbxassetid://139677317950411',
arrowLeftAndLineVerticalAndArrowRight='rbxassetid://76582002086211',
arrowLeftAndRight='rbxassetid://131255998904716',arrowLeftAndRightCircle=
'rbxassetid://92018708920904',arrowLeftAndRightCircleFill=
'rbxassetid://101260287153536',arrowLeftAndRightSquare=
'rbxassetid://106696311729076',arrowLeftAndRightSquareFill=
'rbxassetid://104053611756322',arrowLeftAndRightTextVertical=
'rbxassetid://105731168987658',arrowLeftArrowRight=
'rbxassetid://126907484829097',arrowLeftArrowRightCircle=
'rbxassetid://108565375252584',arrowLeftArrowRightCircleFill=
'rbxassetid://132777863386711',arrowLeftArrowRightSquare=
'rbxassetid://76772045577087',arrowLeftArrowRightSquareFill=
'rbxassetid://137418280059179',arrowLeftCircle='rbxassetid://128580341438564',
arrowLeftCircleDotted='rbxassetid://80098770076714',arrowLeftCircleFill=
'rbxassetid://129347210326426',arrowLeftSquare='rbxassetid://70731607187432',
arrowLeftSquareFill='rbxassetid://77669000354279',arrowLeftToLine=
'rbxassetid://84809777888169',arrowLeftToLineCircle=
'rbxassetid://95667050287745',arrowLeftToLineCircleFill=
'rbxassetid://133685867708333',arrowLeftToLineCompact=
'rbxassetid://108095528571274',arrowLeftToLineSquare=
'rbxassetid://101008265615734',arrowLeftToLineSquareFill=
'rbxassetid://115024811356910',arrowRight='rbxassetid://105335022791801',
arrowRightAndLineVerticalAndArrowLeft='rbxassetid://131068250358492',
arrowRightCircle='rbxassetid://132800764212309',arrowRightCircleDotted=
'rbxassetid://103089050211240',arrowRightCircleFill=
'rbxassetid://91030657053477',arrowRightFilledFilterArrowRight=
'rbxassetid://137853508742382',arrowRightPageOnClipboard=
'rbxassetid://73034395614819',arrowRightSquare='rbxassetid://87365895856110',
arrowRightSquareFill='rbxassetid://118077314182831',arrowRightToLine=
'rbxassetid://106599596369923',arrowRightToLineCircle=
'rbxassetid://132392375458718',arrowRightToLineCircleFill=
'rbxassetid://127030236541102',arrowRightToLineCompact=
'rbxassetid://85675258932891',arrowRightToLineSquare=
'rbxassetid://105804634316858',arrowRightToLineSquareFill=
'rbxassetid://130284453934854',arrowTrianglehead2Clockwise=
'rbxassetid://120004886078638',arrowTrianglehead2ClockwiseRotate90=
'rbxassetid://95004536158964',arrowTrianglehead2ClockwiseRotate90Camera=
'rbxassetid://83773355445722',arrowTrianglehead2ClockwiseRotate90CameraFill=
'rbxassetid://117069297266941',arrowTrianglehead2ClockwiseRotate90Circle=
'rbxassetid://76837546635855',arrowTrianglehead2ClockwiseRotate90CircleFill=
'rbxassetid://125646699018805',arrowTrianglehead2ClockwiseRotate90Icloud=
'rbxassetid://110842292651436',arrowTrianglehead2ClockwiseRotate90IcloudFill=
'rbxassetid://83156429648892',arrowTrianglehead2ClockwiseRotate90PageOnClipboard
='rbxassetid://92313237753356',arrowTrianglehead2Counterclockwise=
'rbxassetid://116637658638058',arrowTrianglehead2CounterclockwiseRotate90=
'rbxassetid://93633635072507',arrowTriangleheadBottomleftCapsulepathClockwise=
'rbxassetid://83435855503662',arrowTriangleheadBranch=
'rbxassetid://119639706252113',arrowTriangleheadClockwise=
'rbxassetid://113903120156035',arrowTriangleheadClockwiseHeart=
'rbxassetid://76711158700076',arrowTriangleheadClockwiseHeartFill=
'rbxassetid://111885492337179',arrowTriangleheadClockwiseIcloud=
'rbxassetid://119325834106076',arrowTriangleheadClockwiseIcloudFill=
'rbxassetid://130817770436829',arrowTriangleheadClockwiseRotate90=
'rbxassetid://125916637513059',arrowTriangleheadCounterclockwise=
'rbxassetid://81134858856836',arrowTriangleheadCounterclockwiseIcloud=
'rbxassetid://97631605490623',arrowTriangleheadCounterclockwiseIcloudFill=
'rbxassetid://124239436659602',arrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://103879478813346',
arrowTriangleheadLeftAndRightRighttriangleLeftRighttriangleRight=
'rbxassetid://95185575126651',
arrowTriangleheadLeftAndRightRighttriangleLeftRighttriangleRightFill=
'rbxassetid://113746416760123',arrowTriangleheadMerge=
'rbxassetid://93660888122611',arrowTriangleheadPull=
'rbxassetid://70848847762237',arrowTriangleheadRectanglepath=
'rbxassetid://122075661819073',arrowTriangleheadSwap=
'rbxassetid://122615966163021',arrowTriangleheadToprightCapsulepathClockwise=
'rbxassetid://87561863258163',arrowTriangleheadTurnUpRight=
'rbxassetid://112409191526627',arrowTriangleheadTurnUpRightCircle=
'rbxassetid://125092376767446',arrowTriangleheadTurnUpRightCircleFill=
'rbxassetid://126793442327731',arrowTriangleheadTurnUpRightDiamond=
'rbxassetid://85736667530199',arrowTriangleheadTurnUpRightDiamondFill=
'rbxassetid://106982346982974',
arrowTriangleheadUpAndDownRighttriangleUpRighttriangleDown=
'rbxassetid://85377495785588',
arrowTriangleheadUpAndDownRighttriangleUpRighttriangleDownFill=
'rbxassetid://123977430406222',arrowTurnDownLeft='rbxassetid://132895074468769',
arrowTurnDownRight='rbxassetid://106816227955900',arrowTurnLeftDown=
'rbxassetid://81993128525415',arrowTurnLeftUp='rbxassetid://72953914184632',
arrowTurnRightDown='rbxassetid://93298093089829',arrowTurnRightUp=
'rbxassetid://82753096109690',arrowTurnUpForwardIphone=
'rbxassetid://126615799314276',arrowTurnUpForwardIphoneFill=
'rbxassetid://94877893999375',arrowTurnUpLeft='rbxassetid://92961717180242',
arrowTurnUpRight='rbxassetid://99344786740317',arrowUp=
'rbxassetid://78648712658851',arrowUpAndDown='rbxassetid://86116303630216',
arrowUpAndDownAndArrowLeftAndRight='rbxassetid://138985551455035',
arrowUpAndDownAndSparkles='rbxassetid://81684341236209',arrowUpAndDownCircle=
'rbxassetid://75916292096750',arrowUpAndDownCircleFill=
'rbxassetid://75127469037390',arrowUpAndDownSquare=
'rbxassetid://114110154035595',arrowUpAndDownSquareFill=
'rbxassetid://75202394337547',arrowUpAndDownTextHorizontal=
'rbxassetid://121347816102410',arrowUpAndLineHorizontalAndArrowDown=
'rbxassetid://105622726329752',arrowUpAndPersonRectanglePortrait=
'rbxassetid://130823180804285',arrowUpAndPersonRectangleTurnLeft=
'rbxassetid://85243691603750',arrowUpAndPersonRectangleTurnRight=
'rbxassetid://105085966547216',arrowUpArrowDown='rbxassetid://113515949560452',
arrowUpArrowDownCircle='rbxassetid://121460592181765',arrowUpArrowDownCircleFill
='rbxassetid://79203277607685',arrowUpArrowDownSquare=
'rbxassetid://137035914755347',arrowUpArrowDownSquareFill=
'rbxassetid://78686762407726',arrowUpBackward='rbxassetid://87724093939497',
arrowUpBackwardAndArrowDownForward='rbxassetid://86635108527984',
arrowUpBackwardAndArrowDownForwardCircle='rbxassetid://115317188393520',
arrowUpBackwardAndArrowDownForwardCircleFill='rbxassetid://77043906356644',
arrowUpBackwardAndArrowDownForwardRectangle='rbxassetid://89847106989116',
arrowUpBackwardAndArrowDownForwardRectangleFill='rbxassetid://114674554996662',
arrowUpBackwardAndArrowDownForwardSquare='rbxassetid://96097826426907',
arrowUpBackwardAndArrowDownForwardSquareFill='rbxassetid://100908924982344',
arrowUpBackwardBottomtrailingRectangle='rbxassetid://132306797856286',
arrowUpBackwardBottomtrailingRectangleFill='rbxassetid://121132174005134',
arrowUpBackwardCircle='rbxassetid://87697874759260',arrowUpBackwardCircleDotted=
'rbxassetid://72881423591822',arrowUpBackwardCircleFill=
'rbxassetid://120193357020420',arrowUpBackwardSquare=
'rbxassetid://71077852083839',arrowUpBackwardSquareFill=
'rbxassetid://93398569903581',arrowUpBin='rbxassetid://106712407901427',
arrowUpBinFill='rbxassetid://124011006230787',arrowUpCircle=
'rbxassetid://94911675961250',arrowUpCircleBadgeClock=
'rbxassetid://125476968673977',arrowUpCircleDotted=
'rbxassetid://111193851076319',arrowUpCircleFill='rbxassetid://111941746337748',
arrowUpDocument='rbxassetid://117304311448174',arrowUpDocumentFill=
'rbxassetid://96587047161399',arrowUpFolder='rbxassetid://119174018910178',
arrowUpFolderFill='rbxassetid://127574661082745',arrowUpForward=
'rbxassetid://108134164414127',arrowUpForwardAndArrowDownBackward=
'rbxassetid://130767770587903',arrowUpForwardAndArrowDownBackwardCircle=
'rbxassetid://107376465009384',arrowUpForwardAndArrowDownBackwardCircleFill=
'rbxassetid://103317843905876',arrowUpForwardAndArrowDownBackwardRectangle=
'rbxassetid://115453529068587',arrowUpForwardAndArrowDownBackwardRectangleFill=
'rbxassetid://79439647053232',arrowUpForwardAndArrowDownBackwardSquare=
'rbxassetid://92612164951517',arrowUpForwardAndArrowDownBackwardSquareFill=
'rbxassetid://70813042781223',arrowUpForwardApp='rbxassetid://112829959970173',
arrowUpForwardAppFill='rbxassetid://91761460498076',
arrowUpForwardBottomleadingRectangle='rbxassetid://134640132914184',
arrowUpForwardBottomleadingRectangleFill='rbxassetid://122314628337723',
arrowUpForwardCircle='rbxassetid://109463155016717',arrowUpForwardCircleDotted=
'rbxassetid://108263917514812',arrowUpForwardCircleFill=
'rbxassetid://77346795770693',arrowUpForwardSquare='rbxassetid://74557125136657'
,arrowUpForwardSquareFill='rbxassetid://119788530957673',arrowUpHeart=
'rbxassetid://81211473277206',arrowUpHeartFill='rbxassetid://112776567008638',
arrowUpLeft='rbxassetid://110082174383245',arrowUpLeftAndArrowDownRight=
'rbxassetid://134847585698412',arrowUpLeftAndArrowDownRightCircle=
'rbxassetid://130371359392714',arrowUpLeftAndArrowDownRightCircleFill=
'rbxassetid://132427830878732',arrowUpLeftAndArrowDownRightRectangle=
'rbxassetid://112294474196779',arrowUpLeftAndArrowDownRightRectangleFill=
'rbxassetid://125360402793191',arrowUpLeftAndArrowDownRightSquare=
'rbxassetid://83415006038142',arrowUpLeftAndArrowDownRightSquareFill=
'rbxassetid://115887822292112',arrowUpLeftAndDownRightAndArrowUpRightAndDownLeft
='rbxassetid://129739556524804',arrowUpLeftAndDownRightMagnifyingglass=
'rbxassetid://137681018745874',arrowUpLeftArrowDownRight=
'rbxassetid://75439123019508',arrowUpLeftArrowDownRightCircle=
'rbxassetid://105643094155741',arrowUpLeftArrowDownRightCircleFill=
'rbxassetid://131547709448660',arrowUpLeftArrowDownRightSquare=
'rbxassetid://85909271280986',arrowUpLeftArrowDownRightSquareFill=
'rbxassetid://99819888059672',arrowUpLeftBottomrightRectangle=
'rbxassetid://91558034145888',arrowUpLeftBottomrightRectangleFill=
'rbxassetid://100287102307407',arrowUpLeftCircle='rbxassetid://121309248454944',
arrowUpLeftCircleDotted='rbxassetid://123064294982538',arrowUpLeftCircleFill=
'rbxassetid://78043906697669',arrowUpLeftSquare='rbxassetid://73015943041948',
arrowUpLeftSquareFill='rbxassetid://102556879413712',arrowUpMessage=
'rbxassetid://94225019561009',arrowUpMessageFill='rbxassetid://82404908556915',
arrowUpPageOnClipboard='rbxassetid://78980645203354',arrowUpRight=
'rbxassetid://138844888875215',arrowUpRightAndArrowDownLeft=
'rbxassetid://114866781443213',arrowUpRightAndArrowDownLeftCircle=
'rbxassetid://132105309061719',arrowUpRightAndArrowDownLeftCircleFill=
'rbxassetid://71412626619086',arrowUpRightAndArrowDownLeftRectangle=
'rbxassetid://73363578686738',arrowUpRightAndArrowDownLeftRectangleFill=
'rbxassetid://91445141227731',arrowUpRightAndArrowDownLeftSquare=
'rbxassetid://86339485660754',arrowUpRightAndArrowDownLeftSquareFill=
'rbxassetid://71919422941284',arrowUpRightBottomleftRectangle=
'rbxassetid://130762630351301',arrowUpRightBottomleftRectangleFill=
'rbxassetid://95630843701625',arrowUpRightCircle='rbxassetid://81069070166818',
arrowUpRightCircleDotted='rbxassetid://97345855116158',arrowUpRightCircleFill=
'rbxassetid://128938699095412',arrowUpRightSquare='rbxassetid://85191059189074',
arrowUpRightSquareFill='rbxassetid://137785035586451',arrowUpRightVideo=
'rbxassetid://90919665747630',arrowUpRightVideoFill=
'rbxassetid://88874555643489',arrowUpSquare='rbxassetid://138634860884120',
arrowUpSquareFill='rbxassetid://120264192976826',arrowUpToLine=
'rbxassetid://88548334886822',arrowUpToLineCircle='rbxassetid://83360733774110',
arrowUpToLineCircleFill='rbxassetid://108275997754139',arrowUpToLineCompact=
'rbxassetid://74427170938065',arrowUpToLineSquare='rbxassetid://137278763514757'
,arrowUpToLineSquareFill='rbxassetid://74700745163970',arrowUpTrash=
'rbxassetid://108944881391784',arrowUpTrashFill='rbxassetid://82802068079538',
arrowUturnBackward='rbxassetid://84932065764436',arrowUturnBackwardCircle=
'rbxassetid://109075671313648',arrowUturnBackwardCircleBadgeEllipsis=
'rbxassetid://102431303478130',arrowUturnBackwardCircleFill=
'rbxassetid://82112623202198',arrowUturnBackwardSquare=
'rbxassetid://91917148305042',arrowUturnBackwardSquareFill=
'rbxassetid://99547665686894',arrowUturnDown='rbxassetid://91557237297615',
arrowUturnDownCircle='rbxassetid://88762138487585',arrowUturnDownCircleFill=
'rbxassetid://125070612634633',arrowUturnDownSquare=
'rbxassetid://82491208720289',arrowUturnDownSquareFill=
'rbxassetid://140476075043878',arrowUturnForward='rbxassetid://125471619268828',
arrowUturnForwardCircle='rbxassetid://123731708760049',
arrowUturnForwardCircleFill='rbxassetid://86954998472846',
arrowUturnForwardSquare='rbxassetid://119862778570014',
arrowUturnForwardSquareFill='rbxassetid://73557234553032',arrowUturnLeft=
'rbxassetid://75706868554870',arrowUturnLeftCircle='rbxassetid://80299087209180'
,arrowUturnLeftCircleBadgeEllipsis='rbxassetid://76418451999894',
arrowUturnLeftCircleFill='rbxassetid://111850081065384',arrowUturnLeftSquare=
'rbxassetid://101145741063035',arrowUturnLeftSquareFill=
'rbxassetid://138477537092319',arrowUturnRight='rbxassetid://85195579204455',
arrowUturnRightCircle='rbxassetid://137184521519714',arrowUturnRightCircleFill=
'rbxassetid://96886515578405',arrowUturnRightSquare=
'rbxassetid://84513404712444',arrowUturnRightSquareFill=
'rbxassetid://87526577867039',arrowUturnUp='rbxassetid://130075543513615',
arrowUturnUpCircle='rbxassetid://75243570930387',arrowUturnUpCircleFill=
'rbxassetid://133907130193679',arrowUturnUpSquare='rbxassetid://124293795803561'
,arrowUturnUpSquareFill='rbxassetid://114646903692155',arrowkeys=
'rbxassetid://107093674674175',arrowkeysDownFilled=
'rbxassetid://139314313798595',arrowkeysFill='rbxassetid://83701399567141',
arrowkeysLeftFilled='rbxassetid://80905415438039',arrowkeysRightFilled=
'rbxassetid://78361096026269',arrowkeysUpFilled='rbxassetid://76100553966528',
arrowshapeBackward='rbxassetid://103946413747803',arrowshapeBackwardCircle=
'rbxassetid://96473706444669',arrowshapeBackwardCircleFill=
'rbxassetid://78185643268471',arrowshapeBackwardFill=
'rbxassetid://137234108178990',arrowshapeBounceForward=
'rbxassetid://104361688592034',arrowshapeBounceForwardFill=
'rbxassetid://133551113789028',arrowshapeBounceRight=
'rbxassetid://77629075456961',arrowshapeBounceRightFill=
'rbxassetid://89650738004209',arrowshapeDown='rbxassetid://97389010768394',
arrowshapeDownCircle='rbxassetid://77919837179930',arrowshapeDownCircleFill=
'rbxassetid://111547932661301',arrowshapeDownFill='rbxassetid://99240071360620',
arrowshapeForward='rbxassetid://88362661062616',arrowshapeForwardCircle=
'rbxassetid://84389407203275',arrowshapeForwardCircleFill=
'rbxassetid://78338063364751',arrowshapeForwardFill=
'rbxassetid://95436585159679',arrowshapeLeft='rbxassetid://98701118720679',
arrowshapeLeftArrowshapeRight='rbxassetid://121937636687253',
arrowshapeLeftArrowshapeRightFill='rbxassetid://133886606502645',
arrowshapeLeftCircle='rbxassetid://71393237208194',arrowshapeLeftCircleFill=
'rbxassetid://136385477516373',arrowshapeLeftFill='rbxassetid://118305835198238'
,arrowshapeRight='rbxassetid://120488785795747',arrowshapeRightCircle=
'rbxassetid://121626085651476',arrowshapeRightCircleFill=
'rbxassetid://76461671929970',arrowshapeRightFill='rbxassetid://70999924264432',
arrowshapeTurnUpBackward='rbxassetid://78881537733143',arrowshapeTurnUpBackward2
='rbxassetid://107092825956456',arrowshapeTurnUpBackward2Circle=
'rbxassetid://137756561495871',arrowshapeTurnUpBackward2CircleFill=
'rbxassetid://135826797352330',arrowshapeTurnUpBackward2Fill=
'rbxassetid://131562286201667',arrowshapeTurnUpBackwardBadgeClock=
'rbxassetid://135028155928594',arrowshapeTurnUpBackwardBadgeClockFill=
'rbxassetid://73450872932149',arrowshapeTurnUpBackwardCircle=
'rbxassetid://71825408205036',arrowshapeTurnUpBackwardCircleFill=
'rbxassetid://111623422888025',arrowshapeTurnUpBackwardFill=
'rbxassetid://109125772896311',arrowshapeTurnUpForward=
'rbxassetid://76816503115863',arrowshapeTurnUpForwardCircle=
'rbxassetid://98602961202921',arrowshapeTurnUpForwardCircleFill=
'rbxassetid://70470114130488',arrowshapeTurnUpForwardFill=
'rbxassetid://92381825435352',arrowshapeTurnUpLeft=
'rbxassetid://108614332890595',arrowshapeTurnUpLeft2=
'rbxassetid://78716969962631',arrowshapeTurnUpLeft2Circle=
'rbxassetid://123519063994380',arrowshapeTurnUpLeft2CircleFill=
'rbxassetid://113915966573413',arrowshapeTurnUpLeft2Fill=
'rbxassetid://86951174248840',arrowshapeTurnUpLeftCircle=
'rbxassetid://73601074472826',arrowshapeTurnUpLeftCircleFill=
'rbxassetid://136740410564323',arrowshapeTurnUpLeftFill=
'rbxassetid://94219456241500',arrowshapeTurnUpRight=
'rbxassetid://106325504068133',arrowshapeTurnUpRightCircle=
'rbxassetid://75919927711551',arrowshapeTurnUpRightCircleFill=
'rbxassetid://106669100525653',arrowshapeTurnUpRightFill=
'rbxassetid://134006299248390',arrowshapeUp='rbxassetid://101998758692791',
arrowshapeUpCircle='rbxassetid://99257685761594',arrowshapeUpCircleFill=
'rbxassetid://96806111160193',arrowshapeUpFill='rbxassetid://88008006362903',
arrowshapeZigzagForward='rbxassetid://129549353799543',
arrowshapeZigzagForwardFill='rbxassetid://106898814741078',arrowshapeZigzagRight
='rbxassetid://103673619735369',arrowshapeZigzagRightFill=
'rbxassetid://130759407110780',arrowtriangleBackward=
'rbxassetid://133521659387505',arrowtriangleBackwardCircle=
'rbxassetid://123787083134802',arrowtriangleBackwardCircleFill=
'rbxassetid://126681849588441',arrowtriangleBackwardFill=
'rbxassetid://111916009128209',arrowtriangleBackwardSquare=
'rbxassetid://140338552398565',arrowtriangleBackwardSquareFill=
'rbxassetid://74805844172505',arrowtriangleDown='rbxassetid://78561539908893',
arrowtriangleDownCircle='rbxassetid://77571811725142',
arrowtriangleDownCircleFill='rbxassetid://105503627251065',arrowtriangleDownFill
='rbxassetid://83161926977947',arrowtriangleDownSquare=
'rbxassetid://138420790717673',arrowtriangleDownSquareFill=
'rbxassetid://91021138374724',arrowtriangleForward='rbxassetid://80300262330695'
,arrowtriangleForwardCircle='rbxassetid://88365381729452',
arrowtriangleForwardCircleFill='rbxassetid://136664536116197',
arrowtriangleForwardFill='rbxassetid://131170560324771',
arrowtriangleForwardSquare='rbxassetid://94267906643716',
arrowtriangleForwardSquareFill='rbxassetid://115669972392572',arrowtriangleLeft=
'rbxassetid://82742410867767',
arrowtriangleLeftAndLineVerticalAndArrowtriangleRight=
'rbxassetid://82250512181706',
arrowtriangleLeftAndLineVerticalAndArrowtriangleRightFill=
'rbxassetid://75120863415970',arrowtriangleLeftCircle=
'rbxassetid://130940545964020',arrowtriangleLeftCircleFill=
'rbxassetid://91597184747625',arrowtriangleLeftFill=
'rbxassetid://100246673935435',arrowtriangleLeftSquare=
'rbxassetid://138937088832435',arrowtriangleLeftSquareFill=
'rbxassetid://78816347161577',arrowtriangleRight='rbxassetid://120603282360061',
arrowtriangleRightAndLineVerticalAndArrowtriangleLeft=
'rbxassetid://125551472926573',
arrowtriangleRightAndLineVerticalAndArrowtriangleLeftFill=
'rbxassetid://137442224732768',arrowtriangleRightCircle=
'rbxassetid://137176667486761',arrowtriangleRightCircleFill=
'rbxassetid://108534551246412',arrowtriangleRightFill=
'rbxassetid://135662872907406',arrowtriangleRightSquare=
'rbxassetid://136702219548567',arrowtriangleRightSquareFill=
'rbxassetid://86559277753252',arrowtriangleUp='rbxassetid://111075032702135',
arrowtriangleUpArrowtriangleDownWindowLeft='rbxassetid://111106358415702',
arrowtriangleUpArrowtriangleDownWindowRight='rbxassetid://119754928116595',
arrowtriangleUpCircle='rbxassetid://135791635467358',arrowtriangleUpCircleFill=
'rbxassetid://77906388596939',arrowtriangleUpFill='rbxassetid://95395937128837',
arrowtriangleUpSquare='rbxassetid://121847518853496',arrowtriangleUpSquareFill=
'rbxassetid://90626000681247',aspectratio='rbxassetid://98517185380471',
aspectratioFill='rbxassetid://102579851046592',asterisk=
'rbxassetid://88871616629375',asteriskCircle='rbxassetid://133395747175893',
asteriskCircleFill='rbxassetid://139028990558441',at=
'rbxassetid://137471691924175',atBadgeMinus='rbxassetid://134623783886552',
atBadgePlus='rbxassetid://135293868255291',atCircle=
'rbxassetid://75936351641093',atCircleFill='rbxassetid://134377411625052',atom=
'rbxassetid://108843899986154',audioJackMono='rbxassetid://122663599339745',
audioJackStereo='rbxassetid://106388714289332',australianFootball=
'rbxassetid://83717210268275',australianFootballCircle=
'rbxassetid://91216938435305',australianFootballCircleFill=
'rbxassetid://119703555005899',australianFootballFill=
'rbxassetid://91615056179585',australiandollarsign=
'rbxassetid://108383329795594',
australiandollarsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://103619465363524',australiandollarsignBankBuilding=
'rbxassetid://112691589231459',australiandollarsignBankBuildingFill=
'rbxassetid://110933280500384',australiandollarsignCircle=
'rbxassetid://138087916288699',australiandollarsignCircleFill=
'rbxassetid://139132160908400',australiandollarsignGaugeChartLefthalfRighthalf=
'rbxassetid://105231666559232',
australiandollarsignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://82562990214305',australiandollarsignRing=
'rbxassetid://107065817248382',australiandollarsignRingDashed=
'rbxassetid://98956564238236',australiandollarsignSquare=
'rbxassetid://105212688231309',australiandollarsignSquareFill=
'rbxassetid://75954055268212',australsign='rbxassetid://137973006716083',
australsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://130848259106826',australsignBankBuilding=
'rbxassetid://94265426732688',australsignBankBuildingFill=
'rbxassetid://123552301428619',australsignCircle='rbxassetid://108380187386176',
australsignCircleFill='rbxassetid://127875262466300',
australsignGaugeChartLefthalfRighthalf='rbxassetid://70536095598981',
australsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://100233677044940',
australsignRing='rbxassetid://88317124938109',australsignRingDashed=
'rbxassetid://97213251184448',australsignSquare='rbxassetid://122311949572306',
australsignSquareFill='rbxassetid://94187640679425',automaticBrakesignal=
'rbxassetid://112399930314317',automaticHeadlightHighBeam=
'rbxassetid://135968366756093',automaticHeadlightHighBeamFill=
'rbxassetid://139626180694517',automaticHeadlightLowBeam=
'rbxassetid://82654718912628',automaticHeadlightLowBeamFill=
'rbxassetid://107178223306087',autostartstop='rbxassetid://77457912590274',
autostartstopSlash='rbxassetid://114435810438487',
autostartstopTrianglebadgeExclamationmark='rbxassetid://72480057980841',avRemote
='rbxassetid://96333083759338',avRemoteFill='rbxassetid://103929453784307',axle2
='rbxassetid://75705790851902',axle2DriveshaftDisengaged=
'rbxassetid://85396555863957',axle2FrontAndRearEngaged=
'rbxassetid://120452693782445',axle2FrontDisengaged=
'rbxassetid://129154377234623',axle2FrontEngaged='rbxassetid://80501466407638',
axle2RearDisengaged='rbxassetid://80324857349654',axle2RearEngaged=
'rbxassetid://132431074188989',axle2RearLock='rbxassetid://75264943176879',
bCircle='rbxassetid://109038084515649',bCircleFill='rbxassetid://75555363720266'
,bSquare='rbxassetid://111811804858729',bSquareFill=
'rbxassetid://128837014096906',backpack='rbxassetid://105195953579349',
backpackCircle='rbxassetid://106407965386940',backpackCircleFill=
'rbxassetid://85475403012001',backpackFill='rbxassetid://131750104401055',
backpackSensorTagRadiowavesLeftAndRight='rbxassetid://128407574860682',
backpackSensorTagRadiowavesLeftAndRightFill='rbxassetid://100870002764956',
backward='rbxassetid://122427584897716',backwardCircle=
'rbxassetid://130281893184037',backwardCircleFill='rbxassetid://95748082741134',
backwardEnd='rbxassetid://73396673985237',backwardEndAlt=
'rbxassetid://118305621946322',backwardEndAltFill='rbxassetid://127445169423377'
,backwardEndCircle='rbxassetid://92798094076866',backwardEndCircleFill=
'rbxassetid://140411571509914',backwardEndFill='rbxassetid://138630184521335',
backwardFill='rbxassetid://81244210367969',backwardFrame=
'rbxassetid://131044827344482',backwardFrameFill='rbxassetid://113681075822442',
badgePlusRadiowavesForward='rbxassetid://80276313107018',
badgePlusRadiowavesRight='rbxassetid://104524636862542',bag=
'rbxassetid://83806619279503',bagBadgeMinus='rbxassetid://88414309827799',
bagBadgePlus='rbxassetid://95221444956038',bagBadgeQuestionmark=
'rbxassetid://98313201554730',bagCircle='rbxassetid://90936649788632',
bagCircleFill='rbxassetid://132136901130565',bagFill=
'rbxassetid://133362226558488',bagFillBadgeMinus='rbxassetid://96219013654249',
bagFillBadgePlus='rbxassetid://101245370544483',bagFillBadgeQuestionmark=
'rbxassetid://108878208237447',bahtsign='rbxassetid://74124698800888',
bahtsignArrowTriangleheadCounterclockwiseRotate90='rbxassetid://73848213936133',
bahtsignBankBuilding='rbxassetid://104523668446152',bahtsignBankBuildingFill=
'rbxassetid://128502368268696',bahtsignCircle='rbxassetid://121720395197210',
bahtsignCircleFill='rbxassetid://128305148458106',
bahtsignGaugeChartLefthalfRighthalf='rbxassetid://132156928740514',
bahtsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://127536612463560',
bahtsignRing='rbxassetid://86639737855130',bahtsignRingDashed=
'rbxassetid://96197994846363',bahtsignSquare='rbxassetid://134719432492683',
bahtsignSquareFill='rbxassetid://97050155065694',balloon=
'rbxassetid://104440599584995',balloon2='rbxassetid://132502412439183',
balloon2Fill='rbxassetid://119048138308879',balloonFill=
'rbxassetid://124491712982295',bandage='rbxassetid://138446349462290',
bandageFill='rbxassetid://80002522608083',banknote=
'rbxassetid://136343080453229',banknoteFill='rbxassetid://116710375951738',
barcode='rbxassetid://117287404870584',barcodeViewfinder=
'rbxassetid://122128534490235',barometer='rbxassetid://72325164390494',baseUnit=
'rbxassetid://100753773581760',baseball='rbxassetid://97571414419713',
baseballCircle='rbxassetid://131671829844062',baseballCircleFill=
'rbxassetid://117106713881204',baseballDiamondBases=
'rbxassetid://111867050801343',baseballDiamondBasesOutsIndicator=
'rbxassetid://93708945096775',baseballFill='rbxassetid://85157186852562',basket=
'rbxassetid://106180348978806',basketFill='rbxassetid://136755744597949',
basketball='rbxassetid://109068809121703',basketballCircle=
'rbxassetid://91774708796019',basketballCircleFill='rbxassetid://72598040742035'
,basketballFill='rbxassetid://72986631471097',bathtub=
'rbxassetid://136389567560332',bathtubFill='rbxassetid://97822181240087',
battery0percent='rbxassetid://109801745750538',battery100percent=
'rbxassetid://92406047854421',battery100percentBolt=
'rbxassetid://139235463865177',battery100percentCircle=
'rbxassetid://100741582191314',battery100percentCircleFill=
'rbxassetid://106944163430713',battery25percent='rbxassetid://72894607867936',
battery50percent='rbxassetid://76241070440884',battery75percent=
'rbxassetid://117533833119206',batteryblock='rbxassetid://121629553249088',
batteryblockFill='rbxassetid://101347307883171',batteryblockSlash=
'rbxassetid://121610081368628',batteryblockSlashFill=
'rbxassetid://74028036721108',batteryblockStack='rbxassetid://115116517381557',
batteryblockStackBadgeSnowflake='rbxassetid://78199827655256',
batteryblockStackBadgeSnowflakeFill='rbxassetid://92623354556534',
batteryblockStackFill='rbxassetid://125795512213710',
batteryblockStackTrianglebadgeExclamationmark='rbxassetid://123811371527725',
batteryblockStackTrianglebadgeExclamationmarkFill='rbxassetid://94880673974554',
beachUmbrella='rbxassetid://72865469806459',beachUmbrellaFill=
'rbxassetid://96407034761761',beatsEarphones='rbxassetid://73621409785573',
beatsFitpro='rbxassetid://117645606044444',beatsFitproChargingcase=
'rbxassetid://117440666717597',beatsFitproChargingcaseFill=
'rbxassetid://71759632509502',beatsFitproLeft='rbxassetid://88418709068625',
beatsFitproRight='rbxassetid://115830286382559',beatsHeadphones=
'rbxassetid://73380840337035',beatsPill='rbxassetid://93439483257162',
beatsPillFill='rbxassetid://95478226585165',beatsPowerbeats=
'rbxassetid://81774880364018',beatsPowerbeats3='rbxassetid://74951439393133',
beatsPowerbeats3Left='rbxassetid://101932876942063',beatsPowerbeats3Right=
'rbxassetid://135752504888114',beatsPowerbeatsLeft='rbxassetid://97445451933143'
,beatsPowerbeatsPro='rbxassetid://127293771112234',beatsPowerbeatsPro2=
'rbxassetid://136525865662552',beatsPowerbeatsPro2Chargingcase=
'rbxassetid://112524730561658',beatsPowerbeatsPro2ChargingcaseFill=
'rbxassetid://82844342134163',beatsPowerbeatsPro2Left=
'rbxassetid://93989999644733',beatsPowerbeatsPro2Right=
'rbxassetid://91812278494548',beatsPowerbeatsProChargingcase=
'rbxassetid://133263655529236',beatsPowerbeatsProChargingcaseFill=
'rbxassetid://99941850096551',beatsPowerbeatsProLeft=
'rbxassetid://70990272965839',beatsPowerbeatsProRight=
'rbxassetid://110928518837662',beatsPowerbeatsRight=
'rbxassetid://106040814246712',beatsSolobuds='rbxassetid://81232374366099',
beatsSolobudsChargingcase='rbxassetid://71797823184677',
beatsSolobudsChargingcaseFill='rbxassetid://101814016626767',beatsSolobudsLeft=
'rbxassetid://71958672695604',beatsSolobudsRight='rbxassetid://96581298863151',
beatsStudiobuds='rbxassetid://111451076752414',beatsStudiobudsChargingcase=
'rbxassetid://73000765400424',beatsStudiobudsChargingcaseFill=
'rbxassetid://109848496275498',beatsStudiobudsLeft=
'rbxassetid://137326830510510',beatsStudiobudsPlus=
'rbxassetid://115554839561360',beatsStudiobudsPlusChargingcase=
'rbxassetid://126239738972103',beatsStudiobudsPlusChargingcaseFill=
'rbxassetid://80657113028988',beatsStudiobudsPlusLeft=
'rbxassetid://75699522736651',beatsStudiobudsPlusRight=
'rbxassetid://114336760507028',beatsStudiobudsRight=
'rbxassetid://95674926492092',bedDouble='rbxassetid://101701657145902',
bedDoubleBadgeCheckmark='rbxassetid://109375568035932',
bedDoubleBadgeCheckmarkFill='rbxassetid://120398540124030',bedDoubleCircle=
'rbxassetid://71145319148752',bedDoubleCircleFill='rbxassetid://92627250811342',
bedDoubleFill='rbxassetid://129538544854607',bell='rbxassetid://133308773796467'
,bellAndWavesLeftAndRight='rbxassetid://121214018166023',
bellAndWavesLeftAndRightFill='rbxassetid://88807057176486',bellBadge=
'rbxassetid://102918378539260',bellBadgeCircle='rbxassetid://96523824553022',
bellBadgeCircleFill='rbxassetid://103345485539554',bellBadgeFill=
'rbxassetid://82918576077557',bellBadgeSlash='rbxassetid://102012144577464',
bellBadgeSlashFill='rbxassetid://86863600694155',bellBadgeWaveform=
'rbxassetid://137731188990608',bellBadgeWaveformFill=
'rbxassetid://91685274004740',bellBadgeWaveformSlash=
'rbxassetid://139347903627441',bellBadgeWaveformSlashFill=
'rbxassetid://121644601155346',bellCircle='rbxassetid://119385063880385',
bellCircleFill='rbxassetid://98518266376193',bellFill=
'rbxassetid://120560073129118',bellSlash='rbxassetid://131140638570899',
bellSlashCircle='rbxassetid://101842323179229',bellSlashCircleFill=
'rbxassetid://115293256505054',bellSlashFill='rbxassetid://104881180476235',
bellSquare='rbxassetid://96391651511460',bellSquareFill=
'rbxassetid://114947742366600',beziercurve='rbxassetid://100185315689380',
bicycle='rbxassetid://81287373353221',bicycleCircle=
'rbxassetid://101135313427799',bicycleCircleFill='rbxassetid://123710817001547',
bicycleSensorTagRadiowavesLeftAndRight='rbxassetid://85373873458550',
bicycleSensorTagRadiowavesLeftAndRightFill='rbxassetid://92824033944913',
binoculars='rbxassetid://82003982039983',binocularsCircle=
'rbxassetid://74444046694959',binocularsCircleFill=
'rbxassetid://105266215858774',binocularsFill='rbxassetid://80941766165614',bird
='rbxassetid://130275772797284',birdCircle='rbxassetid://103178316855292',
birdCircleFill='rbxassetid://117217024861213',birdFill=
'rbxassetid://106346146659683',birthdayCake='rbxassetid://109061723759610',
birthdayCakeFill='rbxassetid://126609361495378',bitcoinsign=
'rbxassetid://85045716322963',
bitcoinsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://139903362520300',bitcoinsignBankBuilding=
'rbxassetid://100134569219475',bitcoinsignBankBuildingFill=
'rbxassetid://130799347858075',bitcoinsignCircle='rbxassetid://129872062077257',
bitcoinsignCircleFill='rbxassetid://125803775641316',
bitcoinsignGaugeChartLefthalfRighthalf='rbxassetid://86873732275506',
bitcoinsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://85495972134697',
bitcoinsignRing='rbxassetid://128221652848711',bitcoinsignRingDashed=
'rbxassetid://71179864949845',bitcoinsignSquare='rbxassetid://135576039906233',
bitcoinsignSquareFill='rbxassetid://120728418357680',blindsHorizontalClosed=
'rbxassetid://101420800756552',blindsHorizontalOpen=
'rbxassetid://129804746319995',blindsVerticalClosed=
'rbxassetid://107585669625798',blindsVerticalOpen='rbxassetid://133263406766885'
,bloodPressureCuff='rbxassetid://121817772408929',
bloodPressureCuffBadgeGaugeWithNeedle='rbxassetid://84032836703049',
bloodPressureCuffBadgeGaugeWithNeedleFill='rbxassetid://124578156567435',
bloodPressureCuffFill='rbxassetid://107378979484859',bold=
'rbxassetid://103508086257150',boldItalicUnderline=
'rbxassetid://125199208998114',boldUnderline='rbxassetid://77310772877118',bolt=
'rbxassetid://112193227421813',boltBadgeAutomatic='rbxassetid://122112691291736'
,boltBadgeAutomaticFill='rbxassetid://140324573909470',boltBadgeCheckmark=
'rbxassetid://99899769150346',boltBadgeCheckmarkFill=
'rbxassetid://127797181436416',boltBadgeClock='rbxassetid://99768130135511',
boltBadgeClockFill='rbxassetid://91456545149386',boltBadgeXmark=
'rbxassetid://94502185545264',boltBadgeXmarkFill='rbxassetid://91288260823755',
boltBatteryblock='rbxassetid://91868076624899',boltBatteryblockFill=
'rbxassetid://133692683566130',boltBrakesignal='rbxassetid://73920694387621',
boltCar='rbxassetid://125466798838914',boltCarCircle=
'rbxassetid://116306626599658',boltCarCircleFill='rbxassetid://101258013169069',
boltCarFill='rbxassetid://114448131958206',boltCircle=
'rbxassetid://87781552660597',boltCircleFill='rbxassetid://131576589857912',
boltFill='rbxassetid://71909592017771',boltHeart='rbxassetid://110674472844382',
boltHeartFill='rbxassetid://87457386765556',boltHorizontal=
'rbxassetid://78421984366839',boltHorizontalCircle='rbxassetid://86052284804870'
,boltHorizontalCircleFill='rbxassetid://82912063382135',boltHorizontalFill=
'rbxassetid://89554007915579',boltHorizontalIcloud='rbxassetid://76349290026078'
,boltHorizontalIcloudFill='rbxassetid://119816421830601',boltHouse=
'rbxassetid://125087333698100',boltHouseFill='rbxassetid://97212832287617',
boltRingClosed='rbxassetid://102748200499647',boltShield=
'rbxassetid://106576035322524',boltShieldFill='rbxassetid://82704335162541',
boltSlash='rbxassetid://115719946769678',boltSlashCircle=
'rbxassetid://92077287655944',boltSlashCircleFill='rbxassetid://83130112629665',
boltSlashFill='rbxassetid://107794328445914',boltSquare=
'rbxassetid://74864226733346',boltSquareFill='rbxassetid://111594600812914',
boltTrianglebadgeExclamationmark='rbxassetid://135701608670001',
boltTrianglebadgeExclamationmarkFill='rbxassetid://80426096092615',bonjour=
'rbxassetid://96862684647565',book='rbxassetid://99242168701957',bookAndWrench=
'rbxassetid://105806071385897',bookAndWrenchFill='rbxassetid://127213912485879',
bookBadgePlus='rbxassetid://139444923393735',bookBadgePlusFill=
'rbxassetid://110708660617586',bookCircle='rbxassetid://72975429086279',
bookCircleFill='rbxassetid://122835733327171',bookClosed=
'rbxassetid://102863733968191',bookClosedCircle='rbxassetid://114853803855969',
bookClosedCircleFill='rbxassetid://85473630292871',bookClosedFill=
'rbxassetid://94972983718665',bookFill='rbxassetid://80798326864179',bookPages=
'rbxassetid://118964813491987',bookPagesFill='rbxassetid://106380748151373',
bookmark='rbxassetid://74160648747460',bookmarkCircle=
'rbxassetid://88464339450728',bookmarkCircleFill='rbxassetid://116856497843645',
bookmarkFill='rbxassetid://98979837416554',bookmarkSlash=
'rbxassetid://130159878454862',bookmarkSlashFill='rbxassetid://108272977689020',
bookmarkSquare='rbxassetid://111204770836057',bookmarkSquareFill=
'rbxassetid://94388247283473',booksVertical='rbxassetid://131690780604844',
booksVerticalCircle='rbxassetid://120107789999419',booksVerticalCircleFill=
'rbxassetid://102387103871516',booksVerticalFill='rbxassetid://118674063613880',
brain='rbxassetid://113890207183352',brainFill='rbxassetid://73855894967261',
brainFilledHeadProfile='rbxassetid://116245786010931',brainHeadProfile=
'rbxassetid://101188959120090',brainHeadProfileFill=
'rbxassetid://82786960924872',brakesignal='rbxassetid://119179613669128',
brakesignalDashed='rbxassetid://129497081013961',brazilianrealsign=
'rbxassetid://126167651986830',
brazilianrealsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://101075452345166',brazilianrealsignBankBuilding=
'rbxassetid://98973047591934',brazilianrealsignBankBuildingFill=
'rbxassetid://75511426648552',brazilianrealsignCircle=
'rbxassetid://97704351842459',brazilianrealsignCircleFill=
'rbxassetid://78800068933179',brazilianrealsignGaugeChartLefthalfRighthalf=
'rbxassetid://81631327262795',
brazilianrealsignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://127207787907253',brazilianrealsignRing=
'rbxassetid://74080520557295',brazilianrealsignRingDashed=
'rbxassetid://95538915598421',brazilianrealsignSquare=
'rbxassetid://125151718943130',brazilianrealsignSquareFill=
'rbxassetid://103532987773414',briefcase='rbxassetid://76302379241787',
briefcaseCircle='rbxassetid://110072070889487',briefcaseCircleFill=
'rbxassetid://131985436861601',briefcaseFill='rbxassetid://103653061652865',
briefcaseSensorTagRadiowavesLeftAndRight='rbxassetid://78511981305968',
briefcaseSensorTagRadiowavesLeftAndRightFill='rbxassetid://79579810164856',
bubble='rbxassetid://77226486416802',bubbleAndPencil=
'rbxassetid://80256643483910',bubbleCircle='rbxassetid://125219383982550',
bubbleCircleFill='rbxassetid://109761907896649',bubbleFill=
'rbxassetid://80953989271043',bubbleLeft='rbxassetid://98539019826711',
bubbleLeftAndBubbleRight='rbxassetid://81405663217056',
bubbleLeftAndBubbleRightFill='rbxassetid://102654503925514',
bubbleLeftAndExclamationmarkBubbleRight='rbxassetid://89966432584239',
bubbleLeftAndExclamationmarkBubbleRightFill='rbxassetid://134483997334229',
bubbleLeftAndTextBubbleRight='rbxassetid://132793650670374',
bubbleLeftAndTextBubbleRightFill='rbxassetid://74625622843211',bubbleLeftCircle=
'rbxassetid://123298231169447',bubbleLeftCircleFill=
'rbxassetid://80558763328280',bubbleLeftFill='rbxassetid://118600153442100',
bubbleMiddleBottom='rbxassetid://95146405413872',bubbleMiddleBottomFill=
'rbxassetid://88431074400146',bubbleMiddleTop='rbxassetid://103163735052752',
bubbleMiddleTopFill='rbxassetid://74247184833387',bubbleRight=
'rbxassetid://109759263505956',bubbleRightCircle='rbxassetid://100274251581597',
bubbleRightCircleFill='rbxassetid://84058503436916',bubbleRightFill=
'rbxassetid://118519568619648',bubblesAndSparkles='rbxassetid://106193796714647'
,bubblesAndSparklesFill='rbxassetid://85576812393251',building=
'rbxassetid://80702293989084',building2='rbxassetid://135138223013381',
building2CropCircle='rbxassetid://115503248658638',building2CropCircleFill=
'rbxassetid://112598319040010',building2Fill='rbxassetid://113360848137363',
buildingColumns='rbxassetid://86341549293525',buildingColumnsCircle=
'rbxassetid://89771684145053',buildingColumnsCircleFill=
'rbxassetid://140466597342735',buildingColumnsFill=
'rbxassetid://135456938295031',buildingFill='rbxassetid://118070055554916',burn=
'rbxassetid://100741266216840',burst='rbxassetid://138219935094738',burstFill=
'rbxassetid://90039414124678',bus='rbxassetid://132462958646640',busDoubledecker
='rbxassetid://112589463325662',busDoubledeckerFill=
'rbxassetid://71921128889600',busFill='rbxassetid://116866000758003',
buttonAngledbottomHorizontalLeft='rbxassetid://87772134018393',
buttonAngledbottomHorizontalLeftFill='rbxassetid://123942133715172',
buttonAngledbottomHorizontalRight='rbxassetid://113637188276134',
buttonAngledbottomHorizontalRightFill='rbxassetid://103324610775001',
buttonAngledtopVerticalLeft='rbxassetid://127310958476000',
buttonAngledtopVerticalLeftFill='rbxassetid://124737537456377',
buttonAngledtopVerticalRight='rbxassetid://72737180265307',
buttonAngledtopVerticalRightFill='rbxassetid://102557397415618',buttonHorizontal
='rbxassetid://89869433999115',buttonHorizontalFill=
'rbxassetid://94554169943088',buttonHorizontalTopPress=
'rbxassetid://79953223577947',buttonHorizontalTopPressFill=
'rbxassetid://132950668501139',buttonProgrammable='rbxassetid://115802779906954'
,buttonProgrammableSquare='rbxassetid://128948305769440',
buttonProgrammableSquareFill='rbxassetid://107393189633512',
buttonRoundedbottomHorizontal='rbxassetid://139688387647553',
buttonRoundedbottomHorizontalFill='rbxassetid://88109415580290',
buttonRoundedtopHorizontal='rbxassetid://87461059045784',
buttonRoundedtopHorizontalFill='rbxassetid://84354403826885',
buttonVerticalLeftPress='rbxassetid://71757710227477',
buttonVerticalLeftPressFill='rbxassetid://93218303828413',
buttonVerticalRightPress='rbxassetid://78432364933892',
buttonVerticalRightPressFill='rbxassetid://91015801699440',cCircle=
'rbxassetid://139431771360926',cCircleFill='rbxassetid://105086158279600',
cSquare='rbxassetid://86392562689553',cSquareFill='rbxassetid://118810898759104'
,cabinet='rbxassetid://117939109046665',cabinetFill=
'rbxassetid://87496272804280',cableCoaxial='rbxassetid://126480673488003',
cableConnector='rbxassetid://76793572656021',cableConnectorHorizontal=
'rbxassetid://128003436998791',cableConnectorSlash=
'rbxassetid://138009988511903',cableConnectorVideo='rbxassetid://81787685713406'
,cablecar='rbxassetid://84703971212857',cablecarFill=
'rbxassetid://81461389662851',calendar='rbxassetid://105164540641328',
calendarAndPerson='rbxassetid://118740780914749',calendarBadge=
'rbxassetid://130252787898462',calendarBadgeCheckmark=
'rbxassetid://89365467298075',calendarBadgeClock='rbxassetid://113445508292256',
calendarBadgeExclamationmark='rbxassetid://91916369222792',calendarBadgeLock=
'rbxassetid://118756663428958',calendarBadgeMinus='rbxassetid://82308953693338',
calendarBadgePlus='rbxassetid://99904240335802',calendarCircle=
'rbxassetid://97923261201903',calendarCircleFill='rbxassetid://134415097437184',
calendarDayTimelineLeading='rbxassetid://92071534684267',
calendarDayTimelineLeadingCircle='rbxassetid://123786645364610',
calendarDayTimelineLeadingCircleFill='rbxassetid://95263183942485',
calendarDayTimelineLeft='rbxassetid://140736157337049',
calendarDayTimelineLeftCircle='rbxassetid://128152730753180',
calendarDayTimelineLeftCircleFill='rbxassetid://101649394856316',
calendarDayTimelineRight='rbxassetid://81354582567686',
calendarDayTimelineRightCircle='rbxassetid://106345972168826',
calendarDayTimelineRightCircleFill='rbxassetid://78552056440331',
calendarDayTimelineTrailing='rbxassetid://98699864273977',
calendarDayTimelineTrailingCircle='rbxassetid://106989432306674',
calendarDayTimelineTrailingCircleFill='rbxassetid://84897746955767',camera=
'rbxassetid://106049677807210',cameraAperture='rbxassetid://93748886326097',
cameraBadgeClock='rbxassetid://96954869359913',cameraBadgeClockFill=
'rbxassetid://129572022257677',cameraBadgeEllipsis='rbxassetid://84933908741123'
,cameraBadgeEllipsisFill='rbxassetid://102950043916587',cameraCircle=
'rbxassetid://136575450217928',cameraCircleFill='rbxassetid://132722961016601',
cameraFill='rbxassetid://105791547551825',cameraFilters=
'rbxassetid://132438444257023',cameraMacro='rbxassetid://90308044332993',
cameraMacroCircle='rbxassetid://96744154304790',cameraMacroCircleFill=
'rbxassetid://78401776068992',cameraMacroSlash='rbxassetid://125432490230233',
cameraMacroSlashCircle='rbxassetid://124019357928403',cameraMacroSlashCircleFill
='rbxassetid://79452825367107',cameraMeteringCenterWeighted=
'rbxassetid://105415399925912',cameraMeteringCenterWeightedAverage=
'rbxassetid://124373772637912',cameraMeteringMatrix=
'rbxassetid://72423091421429',cameraMeteringMultispot=
'rbxassetid://101264837387232',cameraMeteringNone='rbxassetid://108629649478914'
,cameraMeteringPartial='rbxassetid://136722268952658',cameraMeteringSpot=
'rbxassetid://131415808991981',cameraMeteringUnknown=
'rbxassetid://103791550705277',cameraOnRectangle='rbxassetid://113075184701608',
cameraOnRectangleFill='rbxassetid://136334711835785',
cameraSensorTagRadiowavesLeftAndRight='rbxassetid://124176766677378',
cameraSensorTagRadiowavesLeftAndRightFill='rbxassetid://75632082063593',
cameraShutterButton='rbxassetid://90456792215761',cameraShutterButtonFill=
'rbxassetid://84963312707697',cameraViewfinder='rbxassetid://85257199196973',
candybarphone='rbxassetid://91875463862137',capslock=
'rbxassetid://129510873768204',capslockFill='rbxassetid://74070850671559',
capsule='rbxassetid://81831381214147',capsuleBottomhalfFilled=
'rbxassetid://79711489491632',capsuleFill='rbxassetid://101969240257558',
capsuleLefthalfFilled='rbxassetid://120935640012724',capsuleOnCapsule=
'rbxassetid://125962363774490',capsuleOnCapsuleFill=
'rbxassetid://85499949727796',capsuleOnRectangle='rbxassetid://95040820166081',
capsuleOnRectangleFill='rbxassetid://105770621396050',capsulePortrait=
'rbxassetid://99642816241588',capsulePortraitBottomhalfFilled=
'rbxassetid://101845181967705',capsulePortraitFill=
'rbxassetid://134018743561010',capsulePortraitLefthalfFilled=
'rbxassetid://89543355365748',capsulePortraitRighthalfFilled=
'rbxassetid://83813796130615',capsulePortraitTophalfFilled=
'rbxassetid://122165531810954',capsuleRighthalfFilled=
'rbxassetid://75406213303179',capsuleTophalfFilled=
'rbxassetid://117681415719315',captionsBubble='rbxassetid://117417929637325',
captionsBubbleFill='rbxassetid://131809011691174',car=
'rbxassetid://128495454882226',car2='rbxassetid://97578883834004',car2Fill=
'rbxassetid://133721936830795',carBadgeGearshape='rbxassetid://121514836966717',
carBadgeGearshapeFill='rbxassetid://94538505333092',carCircle=
'rbxassetid://93535532040070',carCircleFill='rbxassetid://116413251835553',
carFerry='rbxassetid://126066171893470',carFerryFill=
'rbxassetid://115970310065091',carFill='rbxassetid://120157979001146',
carFrontWavesDown='rbxassetid://73974605458298',carFrontWavesDownFill=
'rbxassetid://112916365434291',carFrontWavesLeftAndRightAndUp=
'rbxassetid://93776100100461',carFrontWavesLeftAndRightAndUpFill=
'rbxassetid://100684279352347',carFrontWavesUp='rbxassetid://95563447987178',
carFrontWavesUpFill='rbxassetid://113013581989612',carRear=
'rbxassetid://106736790731856',carRearAndCollisionRoadLane=
'rbxassetid://73279442408907',carRearAndCollisionRoadLaneSlash=
'rbxassetid://89753862961614',carRearAndTireMarks='rbxassetid://90284918615637',
carRearAndTireMarksOff='rbxassetid://121559243559396',carRearAndTireMarksSlash=
'rbxassetid://133003297173007',carRearFill='rbxassetid://94875494313831',
carRearHazardsign='rbxassetid://81176038582242',carRearHazardsignFill=
'rbxassetid://118042380942420',carRearRoadLane='rbxassetid://106146900593351',
carRearRoadLaneDashed='rbxassetid://72003677424945',
carRearRoadLaneDashedArrowtriangle2Outward='rbxassetid://131007235242118',
carRearRoadLaneDistance1='rbxassetid://127940734523444',
carRearRoadLaneDistance1AndGaugeOpenWithLinesNeedle67percentAndArrowtriangle=
'rbxassetid://94334222817359',carRearRoadLaneDistance2=
'rbxassetid://71174865675916',
carRearRoadLaneDistance2AndGaugeOpenWithLinesNeedle67percentAndArrowtriangle=
'rbxassetid://119164907256182',carRearRoadLaneDistance3=
'rbxassetid://135647159201253',
carRearRoadLaneDistance3AndGaugeOpenWithLinesNeedle67percentAndArrowtriangle=
'rbxassetid://76305713845211',carRearRoadLaneDistance4=
'rbxassetid://102603761274028',
carRearRoadLaneDistance4AndGaugeOpenWithLinesNeedle67percentAndArrowtriangle=
'rbxassetid://105468736067403',carRearRoadLaneDistance5=
'rbxassetid://112963693265764',
carRearRoadLaneDistance5AndGaugeOpenWithLinesNeedle67percentAndArrowtriangle=
'rbxassetid://107369882581908',carRearRoadLaneOff='rbxassetid://109767288290225'
,carRearRoadLaneWaveUp='rbxassetid://100245066913455',
carRearTiltRoadLanesCurvedRight='rbxassetid://127430126262647',carRearWavesUp=
'rbxassetid://129534910434287',carRearWavesUpFill='rbxassetid://132630790194670'
,carSide='rbxassetid://123727230876023',carSideAirCirculate=
'rbxassetid://129819788394181',carSideAirCirculateFill=
'rbxassetid://84434233331006',carSideAirFresh='rbxassetid://96826195272673',
carSideAirFreshFill='rbxassetid://119969844288900',carSideAndExclamationmark=
'rbxassetid://131867351843012',carSideAndExclamationmarkFill=
'rbxassetid://114889185254237',carSideArrowLeftAndRight=
'rbxassetid://132025369793047',carSideArrowLeftAndRightFill=
'rbxassetid://114322757158315',carSideArrowtriangleDown=
'rbxassetid://70559673683673',carSideArrowtriangleDownFill=
'rbxassetid://123699022550270',carSideArrowtriangleUp=
'rbxassetid://86416823330109',carSideArrowtriangleUpArrowtriangleDown=
'rbxassetid://136484069843029',carSideArrowtriangleUpArrowtriangleDownFill=
'rbxassetid://112039352485090',carSideArrowtriangleUpFill=
'rbxassetid://101950075430741',carSideFill='rbxassetid://134365111379084',
carSideFrontOpen='rbxassetid://113954965550455',carSideFrontOpenCrop=
'rbxassetid://82479251905391',carSideFrontOpenCropFill=
'rbxassetid://104032074636855',carSideFrontOpenFill=
'rbxassetid://84063705179151',carSideHillDescentControl=
'rbxassetid://134307743041436',carSideHillDescentControlFill=
'rbxassetid://70571506563107',carSideHillDown='rbxassetid://127663646775564',
carSideHillDownAndGaugeOpenWithLinesNeedle25percentAndArrowtriangle=
'rbxassetid://135438547755627',
carSideHillDownAndGaugeOpenWithLinesNeedle25percentAndArrowtriangleFill=
'rbxassetid://126863447170500',carSideHillDownFill=
'rbxassetid://123845478959576',carSideHillUp='rbxassetid://85662738311671',
carSideHillUpFill='rbxassetid://137182275581929',carSideLock=
'rbxassetid://89133577708824',carSideLockFill='rbxassetid://139374257452667',
carSideLockOpen='rbxassetid://73799606963227',carSideLockOpenFill=
'rbxassetid://113815388443315',carSideRearAndCollisionAndCarSideFront=
'rbxassetid://99933366959957',
carSideRearAndCollisionAndCarSideFrontAndArrowForward=
'rbxassetid://131674539097286',
carSideRearAndCollisionAndCarSideFrontAndSteeringwheel=
'rbxassetid://95675889148537',carSideRearAndCollisionAndCarSideFrontSlash=
'rbxassetid://126195085802057',carSideRearAndExclamationmarkAndCarSideFront=
'rbxassetid://140644239188800',carSideRearAndExclamationmarkAndCarSideFrontOff=
'rbxassetid://138499860628048',carSideRearAndWave3AndCarSideFront=
'rbxassetid://71923913494011',carSideRearCropTrunkPartition=
'rbxassetid://84550560822967',carSideRearCropTrunkPartitionFill=
'rbxassetid://123573397918828',carSideRearOpen='rbxassetid://88012230238253',
carSideRearOpenCrop='rbxassetid://129882870954526',carSideRearOpenCropFill=
'rbxassetid://126796702854347',carSideRearOpenFill='rbxassetid://86788038143818'
,carSideRearTowHitch='rbxassetid://134168233048942',carSideRearTowHitchFill=
'rbxassetid://86801046249588',carSideRoofCargoCarrier=
'rbxassetid://104771078663560',carSideRoofCargoCarrierFill=
'rbxassetid://90138154580521',carSideRoofCargoCarrierSlash=
'rbxassetid://138727479392520',carSideRoofCargoCarrierSlashFill=
'rbxassetid://92406618590649',carTopArrowtriangleFrontLeft=
'rbxassetid://123222259486467',carTopArrowtriangleFrontLeftFill=
'rbxassetid://84759596863076',carTopArrowtriangleFrontRight=
'rbxassetid://117838105385476',carTopArrowtriangleFrontRightFill=
'rbxassetid://92362685636289',carTopArrowtriangleRearLeft=
'rbxassetid://73954649933688',carTopArrowtriangleRearLeftFill=
'rbxassetid://132436036528631',carTopArrowtriangleRearRight=
'rbxassetid://78772570541846',carTopArrowtriangleRearRightFill=
'rbxassetid://108137783518479',
carTopDoorFrontLeftAndFrontRightAndRearLeftAndRearRightOpen=
'rbxassetid://80363013214098',
carTopDoorFrontLeftAndFrontRightAndRearLeftAndRearRightOpenFill=
'rbxassetid://108894171052291',carTopDoorFrontLeftAndFrontRightAndRearLeftOpen=
'rbxassetid://123212226339031',
carTopDoorFrontLeftAndFrontRightAndRearLeftOpenFill=
'rbxassetid://101562090992305',carTopDoorFrontLeftAndFrontRightAndRearRightOpen=
'rbxassetid://110631034665291',
carTopDoorFrontLeftAndFrontRightAndRearRightOpenFill=
'rbxassetid://139873570903173',carTopDoorFrontLeftAndFrontRightOpen=
'rbxassetid://74788648223269',carTopDoorFrontLeftAndFrontRightOpenFill=
'rbxassetid://86215017441713',carTopDoorFrontLeftAndRearLeftAndRearRightOpen=
'rbxassetid://131484052357343',
carTopDoorFrontLeftAndRearLeftAndRearRightOpenFill='rbxassetid://89363125622972'
,carTopDoorFrontLeftAndRearLeftOpen='rbxassetid://136031029179173',
carTopDoorFrontLeftAndRearLeftOpenFill='rbxassetid://137046170507650',
carTopDoorFrontLeftAndRearRightOpen='rbxassetid://87031066098040',
carTopDoorFrontLeftAndRearRightOpenFill='rbxassetid://128332791898221',
carTopDoorFrontLeftOpen='rbxassetid://81445878346757',
carTopDoorFrontLeftOpenFill='rbxassetid://109057462580081',
carTopDoorFrontRightAndRearLeftAndRearRightOpen='rbxassetid://78018818420040',
carTopDoorFrontRightAndRearLeftAndRearRightOpenFill=
'rbxassetid://84749548678375',carTopDoorFrontRightAndRearLeftOpen=
'rbxassetid://120269651846295',carTopDoorFrontRightAndRearLeftOpenFill=
'rbxassetid://78066029950386',carTopDoorFrontRightAndRearRightOpen=
'rbxassetid://70567122240859',carTopDoorFrontRightAndRearRightOpenFill=
'rbxassetid://116264095664357',carTopDoorFrontRightOpen=
'rbxassetid://97417623414623',carTopDoorFrontRightOpenFill=
'rbxassetid://84331480606171',carTopDoorRearLeftAndRearRightOpen=
'rbxassetid://89308026145118',carTopDoorRearLeftAndRearRightOpenFill=
'rbxassetid://110224159210139',carTopDoorRearLeftOpen=
'rbxassetid://117679988412307',carTopDoorRearLeftOpenFill=
'rbxassetid://103275469741774',carTopDoorRearRightOpen=
'rbxassetid://134727516346244',carTopDoorRearRightOpenFill=
'rbxassetid://76042471844419',carTopDoorSlidingLeftOpen=
'rbxassetid://114193700451377',carTopDoorSlidingLeftOpenFill=
'rbxassetid://88845688904454',carTopDoorSlidingRightOpen=
'rbxassetid://125155196045524',carTopDoorSlidingRightOpenFill=
'rbxassetid://90631209953802',
carTopFrontRadiowavesFrontLeftAndFrontAndFrontRight=
'rbxassetid://116813641912560',
carTopFrontRadiowavesFrontLeftAndFrontAndFrontRightFill=
'rbxassetid://99204992957472',carTopLaneDashedArrowtriangleInward=
'rbxassetid://111343898744983',carTopLaneDashedArrowtriangleInwardFill=
'rbxassetid://75649906069159',carTopLaneDashedBadgeSteeringwheel=
'rbxassetid://109175591331591',carTopLaneDashedBadgeSteeringwheelFill=
'rbxassetid://99737560328566',carTopLaneDashedDepartureLeft=
'rbxassetid://105946066504996',carTopLaneDashedDepartureLeftFill=
'rbxassetid://133030233235812',carTopLaneDashedDepartureLeftSlash=
'rbxassetid://102591664146154',carTopLaneDashedDepartureLeftSlashFill=
'rbxassetid://134823708023730',carTopLaneDashedDepartureRight=
'rbxassetid://128417056182752',carTopLaneDashedDepartureRightFill=
'rbxassetid://117041928860784',carTopLaneDashedDepartureRightSlash=
'rbxassetid://118261490125488',carTopLaneDashedDepartureRightSlashFill=
'rbxassetid://130656050187719',carTopRadiowaves2FrontLeftFrontFrontRight=
'rbxassetid://95526058568755',carTopRadiowaves2FrontLeftFrontFrontRightFill=
'rbxassetid://77293070849440',carTopRadiowaves2RearLeftRearRearRight=
'rbxassetid://115935038426649',carTopRadiowaves2RearLeftRearRearRightFill=
'rbxassetid://82056639801498',carTopRadiowavesFront=
'rbxassetid://133607318826630',carTopRadiowavesFrontFill=
'rbxassetid://81383834313147',carTopRadiowavesRear=
'rbxassetid://125297507525835',carTopRadiowavesRearFill=
'rbxassetid://78203718067568',carTopRadiowavesRearLeft=
'rbxassetid://80414529165992',carTopRadiowavesRearLeftAndRearRight=
'rbxassetid://106289875711195',carTopRadiowavesRearLeftAndRearRightFill=
'rbxassetid://105696129533991',carTopRadiowavesRearLeftCarTopFront=
'rbxassetid://76064734358653',carTopRadiowavesRearLeftCarTopFrontFill=
'rbxassetid://97770197457840',carTopRadiowavesRearLeftFill=
'rbxassetid://108315739511697',carTopRadiowavesRearRight=
'rbxassetid://75242343694458',carTopRadiowavesRearRightBadgeExclamationmark=
'rbxassetid://139745112794803',carTopRadiowavesRearRightBadgeExclamationmarkFill
='rbxassetid://83075965159111',carTopRadiowavesRearRightBadgeXmark=
'rbxassetid://123230437909232',carTopRadiowavesRearRightBadgeXmarkFill=
'rbxassetid://70407895765158',carTopRadiowavesRearRightCarTopFront=
'rbxassetid://131947111049560',carTopRadiowavesRearRightCarTopFrontFill=
'rbxassetid://72077613330190',carTopRadiowavesRearRightFill=
'rbxassetid://88012174287090',carTopRearRadiowavesRearLeftAndRearAndRearRight=
'rbxassetid://88120058138309',
carTopRearRadiowavesRearLeftAndRearAndRearRightFill=
'rbxassetid://73216347350191',carTopVideoRearLeft='rbxassetid://72271021597106',
carTopVideoRearLeftFill='rbxassetid://71654523784960',carTopVideoRearRight=
'rbxassetid://114116872591998',carTopVideoRearRightFill=
'rbxassetid://137959831857002',carWindowLeft='rbxassetid://86756481448690',
carWindowLeftBadgeExclamationmark='rbxassetid://128521824982284',
carWindowLeftBadgeLock='rbxassetid://74945260791121',carWindowLeftBadgeXmark=
'rbxassetid://126631257556393',carWindowLeftExclamationmark=
'rbxassetid://73994718634793',carWindowLeftXmark='rbxassetid://99106662522022',
carWindowRight='rbxassetid://95397796099425',carWindowRightBadgeExclamationmark=
'rbxassetid://137691133108353',carWindowRightBadgeLock=
'rbxassetid://109825199930596',carWindowRightBadgeXmark=
'rbxassetid://113338395744934',carWindowRightExclamationmark=
'rbxassetid://98295643301246',carWindowRightXmark='rbxassetid://108600675223720'
,carbonDioxideCloud='rbxassetid://76366963920532',carbonDioxideCloudFill=
'rbxassetid://134345598862121',carbonMonoxideCloud=
'rbxassetid://116628010954326',carbonMonoxideCloudFill=
'rbxassetid://108732016966215',carrot='rbxassetid://86588469030196',carrotFill=
'rbxassetid://117181669639260',carseatLeft='rbxassetid://92701856050395',
carseatLeft1='rbxassetid://130693652944034',carseatLeft1Fill=
'rbxassetid://106012772377881',carseatLeft2='rbxassetid://97305403792548',
carseatLeft2Fill='rbxassetid://85685205406693',carseatLeft3=
'rbxassetid://127225716289512',carseatLeft3Fill='rbxassetid://98079963311844',
carseatLeftAndHeatWaves='rbxassetid://136058084732653',
carseatLeftAndHeatWavesFill='rbxassetid://138059967999511',
carseatLeftBackrestUpAndDown='rbxassetid://108455978558375',
carseatLeftBackrestUpAndDownFill='rbxassetid://132193352969179',carseatLeftFan=
'rbxassetid://109491477716325',carseatLeftFanFill='rbxassetid://92653087286034',
carseatLeftFill='rbxassetid://118896758627771',carseatLeftForwardAndBackward=
'rbxassetid://103737852803606',carseatLeftForwardAndBackwardFill=
'rbxassetid://80914430111661',carseatLeftMassage='rbxassetid://110277390360496',
carseatLeftMassageFill='rbxassetid://113778384545082',carseatLeftUpAndDown=
'rbxassetid://139435554215192',carseatLeftUpAndDownFill=
'rbxassetid://93609533999754',carseatRight='rbxassetid://139995356802592',
carseatRight1='rbxassetid://72264832504032',carseatRight1Fill=
'rbxassetid://137548535535010',carseatRight2='rbxassetid://125971106368177',
carseatRight2Fill='rbxassetid://140561721922196',carseatRight3=
'rbxassetid://117423139105628',carseatRight3Fill='rbxassetid://92691917532826',
carseatRightAndHeatWaves='rbxassetid://131216200983766',
carseatRightAndHeatWavesFill='rbxassetid://135515163923925',
carseatRightBackrestUpAndDown='rbxassetid://116883448300446',
carseatRightBackrestUpAndDownFill='rbxassetid://110724862492710',carseatRightFan
='rbxassetid://72695279927845',carseatRightFanFill=
'rbxassetid://136482229464042',carseatRightFill='rbxassetid://118564586970353',
carseatRightForwardAndBackward='rbxassetid://133538142433603',
carseatRightForwardAndBackwardFill='rbxassetid://130117771768091',
carseatRightMassage='rbxassetid://97404558707824',carseatRightMassageFill=
'rbxassetid://124278125047452',carseatRightUpAndDown=
'rbxassetid://103900898320263',carseatRightUpAndDownFill=
'rbxassetid://81259033424146',cart='rbxassetid://120970975539765',cartBadgeClock
='rbxassetid://121047592354792',cartBadgeClockFill=
'rbxassetid://108098854699224',cartBadgeMinus='rbxassetid://94921095508486',
cartBadgePlus='rbxassetid://74497004572470',cartBadgeQuestionmark=
'rbxassetid://99604576439516',cartCircle='rbxassetid://74390949952121',
cartCircleFill='rbxassetid://81723599390977',cartFill=
'rbxassetid://122095585357706',cartFillBadgeMinus='rbxassetid://78126067125241',
cartFillBadgePlus='rbxassetid://109643096003552',cartFillBadgeQuestionmark=
'rbxassetid://96329797572754',case='rbxassetid://131671593455396',caseFill=
'rbxassetid://133667743842833',cat='rbxassetid://76438043402065',catCircle=
'rbxassetid://125040552553301',catCircleFill='rbxassetid://105496505191189',
catFill='rbxassetid://75596623477851',cedisign='rbxassetid://103299702073207',
cedisignArrowTriangleheadCounterclockwiseRotate90='rbxassetid://110438377324030'
,cedisignBankBuilding='rbxassetid://94391063534756',cedisignBankBuildingFill=
'rbxassetid://98460480681983',cedisignCircle='rbxassetid://130372607949366',
cedisignCircleFill='rbxassetid://92011821597293',
cedisignGaugeChartLefthalfRighthalf='rbxassetid://130943895529460',
cedisignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://91190898873471',
cedisignRing='rbxassetid://130183023961084',cedisignRingDashed=
'rbxassetid://107629417082778',cedisignSquare='rbxassetid://138180702536840',
cedisignSquareFill='rbxassetid://111554398707505',cellularbars=
'rbxassetid://125832308574063',cellularbarsCircle='rbxassetid://106772243084439'
,cellularbarsCircleFill='rbxassetid://75137148416999',centsign=
'rbxassetid://105925251377949',centsignArrowTriangleheadCounterclockwiseRotate90
='rbxassetid://116762184007471',centsignBankBuilding=
'rbxassetid://97825181867350',centsignBankBuildingFill=
'rbxassetid://127430294496635',centsignCircle='rbxassetid://82940891704153',
centsignCircleFill='rbxassetid://90090839272956',
centsignGaugeChartLefthalfRighthalf='rbxassetid://114208104708871',
centsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://129300951818917',
centsignRing='rbxassetid://140362456321464',centsignRingDashed=
'rbxassetid://74787978516657',centsignSquare='rbxassetid://111622219613255',
centsignSquareFill='rbxassetid://124455198437315',chair=
'rbxassetid://81031147056732',chairFill='rbxassetid://104928042194690',
chairLounge='rbxassetid://86782375467651',chairLoungeFill=
'rbxassetid://134838176918598',chandelier='rbxassetid://114256133376087',
chandelierFill='rbxassetid://140034611273760',character=
'rbxassetid://70520208661164',characterBookClosed='rbxassetid://98811544621143',
characterBookClosedFill='rbxassetid://106951098218047',characterBubble=
'rbxassetid://135972436057073',characterBubbleFill='rbxassetid://88670019070119'
,characterCircle='rbxassetid://120600115126394',characterCircleFill=
'rbxassetid://138588485044134',characterCursorIbeam=
'rbxassetid://70471184047263',characterDuployan='rbxassetid://126945262264539',
characterMagnify='rbxassetid://93955130363457',characterPhonetic=
'rbxassetid://115826427725683',characterSquare='rbxassetid://74845204135026',
characterSquareFill='rbxassetid://73662884801491',characterSutton=
'rbxassetid://80770715140920',characterTextJustify='rbxassetid://95949746183744'
,characterTextbox='rbxassetid://111364553064358',characterTextboxBadgeSparkles=
'rbxassetid://131808194478683',charactersLowercase='rbxassetid://88128764384529'
,charactersUppercase='rbxassetid://71028973098762',chartBar=
'rbxassetid://97312304809598',chartBarFill='rbxassetid://71315621261538',
chartBarHorizontalPage='rbxassetid://104809785659828',chartBarHorizontalPageFill
='rbxassetid://79906850504673',chartBarXaxis='rbxassetid://70369400252147',
chartBarXaxisAscending='rbxassetid://82359236009090',
chartBarXaxisAscendingBadgeClock='rbxassetid://124185325005143',
chartBarXaxisDescending='rbxassetid://133906778698960',chartBarYaxis=
'rbxassetid://99099708214794',chartDotsScatter='rbxassetid://121018735962977',
chartLineDowntrendXyaxis='rbxassetid://72084011691095',
chartLineDowntrendXyaxisCircle='rbxassetid://76215613064769',
chartLineDowntrendXyaxisCircleFill='rbxassetid://138273764461717',
chartLineFlattrendXyaxis='rbxassetid://110690574915489',
chartLineFlattrendXyaxisCircle='rbxassetid://76783856599745',
chartLineFlattrendXyaxisCircleFill='rbxassetid://132276486757230',
chartLineTextClipboard='rbxassetid://97981178714527',chartLineTextClipboardFill=
'rbxassetid://96087723608983',chartLineUptrendXyaxis=
'rbxassetid://139061824225089',chartLineUptrendXyaxisCircle=
'rbxassetid://96828643866957',chartLineUptrendXyaxisCircleFill=
'rbxassetid://80941846198870',chartPie='rbxassetid://122200783940691',
chartPieFill='rbxassetid://128608698529987',chartXyaxisLine=
'rbxassetid://92614224702926',checklist='rbxassetid://125339100043044',
checklistChecked='rbxassetid://128970854124435',checklistUnchecked=
'rbxassetid://95294773465186',checkmark='rbxassetid://117709091345748',
checkmarkApp='rbxassetid://73945071177652',checkmarkAppFill=
'rbxassetid://96250250366221',checkmarkApplewatch='rbxassetid://97180578736402',
checkmarkArrowTriangleheadClockwise='rbxassetid://107495937127772',
checkmarkArrowTriangleheadCounterclockwise='rbxassetid://103676257085698',
checkmarkBubble='rbxassetid://113427758353101',checkmarkBubbleFill=
'rbxassetid://138390397309804',checkmarkCircle='rbxassetid://113497182715811',
checkmarkCircleBadgeAirplane='rbxassetid://128585382975580',
checkmarkCircleBadgeAirplaneFill='rbxassetid://125423417750617',
checkmarkCircleBadgePlus='rbxassetid://117765136842107',
checkmarkCircleBadgePlusFill='rbxassetid://135313130011967',
checkmarkCircleBadgeQuestionmark='rbxassetid://138820692867130',
checkmarkCircleBadgeQuestionmarkFill='rbxassetid://95057426558621',
checkmarkCircleBadgeXmark='rbxassetid://81262261190949',
checkmarkCircleBadgeXmarkFill='rbxassetid://103457398237353',
checkmarkCircleDotted='rbxassetid://95176388046983',checkmarkCircleFill=
'rbxassetid://80701432625608',checkmarkCircleTrianglebadgeExclamationmark=
'rbxassetid://70460675920599',checkmarkCircleTrianglebadgeExclamationmarkFill=
'rbxassetid://70825651489467',checkmarkDiamond='rbxassetid://127964709086707',
checkmarkDiamondFill='rbxassetid://126380432099282',checkmarkIcloud=
'rbxassetid://91002425672304',checkmarkIcloudFill='rbxassetid://118772663634007'
,checkmarkMessage='rbxassetid://115941539799893',checkmarkMessageFill=
'rbxassetid://106074683578973',checkmarkRectangle='rbxassetid://80011475108827',
checkmarkRectangleFill='rbxassetid://109346468847413',checkmarkRectanglePortrait
='rbxassetid://110710722715340',checkmarkRectanglePortraitFill=
'rbxassetid://106934167289889',checkmarkRectangleStack=
'rbxassetid://105010458802288',checkmarkRectangleStackFill=
'rbxassetid://101287404944931',checkmarkSeal='rbxassetid://79347843080910',
checkmarkSealFill='rbxassetid://108989677395192',checkmarkSealTextPage=
'rbxassetid://95930917032294',checkmarkSealTextPageFill=
'rbxassetid://81864860271766',checkmarkShield='rbxassetid://116476478760645',
checkmarkShieldFill='rbxassetid://123693100854806',checkmarkSquare=
'rbxassetid://90140343305408',checkmarkSquareFill='rbxassetid://81712190011585',
chevronBackward='rbxassetid://91193161432296',chevronBackward2=
'rbxassetid://139857646802844',chevronBackwardChevronBackwardDotted=
'rbxassetid://134698694323812',chevronBackwardCircle=
'rbxassetid://111843022069820',chevronBackwardCircleFill=
'rbxassetid://106726041209256',chevronBackwardSquare=
'rbxassetid://119395442945672',chevronBackwardSquareFill=
'rbxassetid://137650555775550',chevronBackwardToLine=
'rbxassetid://119263941294390',chevronCompactBackward=
'rbxassetid://133203717156144',chevronCompactDown='rbxassetid://100092606470558'
,chevronCompactForward='rbxassetid://71064478971121',chevronCompactLeft=
'rbxassetid://102340620945726',chevronCompactLeftChevronCompactRight=
'rbxassetid://74422735550656',chevronCompactRight='rbxassetid://90848510900151',
chevronCompactUp='rbxassetid://129845251522502',
chevronCompactUpChevronCompactDown='rbxassetid://101290677380772',
chevronCompactUpChevronCompactRightChevronCompactDownChevronCompactLeft=
'rbxassetid://94444191245039',chevronDown='rbxassetid://116067852952871',
chevronDown2='rbxassetid://83293178829681',chevronDownCircle=
'rbxassetid://106048568785334',chevronDownCircleFill=
'rbxassetid://137482091804957',chevronDownDotted2='rbxassetid://125100120340183'
,chevronDownForward2='rbxassetid://80146358470761',chevronDownForwardDotted2=
'rbxassetid://105001073803635',chevronDownRight2='rbxassetid://117900088200237',
chevronDownRightDotted2='rbxassetid://70778643695183',chevronDownSquare=
'rbxassetid://90429530228079',chevronDownSquareFill=
'rbxassetid://113144434425347',chevronForward='rbxassetid://85597205001941',
chevronForward2='rbxassetid://118305569049773',chevronForwardCircle=
'rbxassetid://85622575012397',chevronForwardCircleFill=
'rbxassetid://105928949880596',chevronForwardDottedChevronForward=
'rbxassetid://118611532406974',chevronForwardSquare=
'rbxassetid://121248135678052',chevronForwardSquareFill=
'rbxassetid://136901211975528',chevronForwardToLine=
'rbxassetid://113251427810828',chevronLeft='rbxassetid://120112805068615',
chevronLeft2='rbxassetid://135271611992131',chevronLeftChevronLeftDotted=
'rbxassetid://89493654076591',chevronLeftChevronRight=
'rbxassetid://126091837718448',chevronLeftCircle='rbxassetid://79371642062729',
chevronLeftCircleFill='rbxassetid://85025727393821',
chevronLeftForwardslashChevronRight='rbxassetid://89806716346278',
chevronLeftSquare='rbxassetid://76844555323813',chevronLeftSquareFill=
'rbxassetid://85373254243633',chevronLeftToLine='rbxassetid://110571725143380',
chevronRight='rbxassetid://78578344679238',chevronRight2=
'rbxassetid://128617068570209',chevronRightCircle='rbxassetid://137319086285901'
,chevronRightCircleFill='rbxassetid://106247315510084',
chevronRightDottedChevronRight='rbxassetid://90280104847900',chevronRightSquare=
'rbxassetid://108383056663215',chevronRightSquareFill=
'rbxassetid://112928454370933',chevronRightToLine='rbxassetid://88538085601212',
chevronUp='rbxassetid://71994936177602',chevronUp2='rbxassetid://79472666988214'
,chevronUpChevronDown='rbxassetid://115187614425058',chevronUpChevronDownSquare=
'rbxassetid://90721207484341',chevronUpChevronDownSquareFill=
'rbxassetid://77237411567916',chevronUpChevronRightChevronDownChevronLeft=
'rbxassetid://76064929254887',chevronUpCircle='rbxassetid://100279286464112',
chevronUpCircleFill='rbxassetid://116618139465058',chevronUpDotted2=
'rbxassetid://104672280488626',chevronUpForward2='rbxassetid://104274296097003',
chevronUpForwardDotted2='rbxassetid://95660075484148',chevronUpRight2=
'rbxassetid://109081501210155',chevronUpRightDotted2=
'rbxassetid://81120090509930',chevronUpSquare='rbxassetid://128957757104189',
chevronUpSquareFill='rbxassetid://139591043142298',chineseyuanrenminbisign=
'rbxassetid://106647652850967',
chineseyuanrenminbisignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://133627043954478',chineseyuanrenminbisignBankBuilding=
'rbxassetid://135345149638301',chineseyuanrenminbisignBankBuildingFill=
'rbxassetid://117468270794690',chineseyuanrenminbisignCircle=
'rbxassetid://99238289619208',chineseyuanrenminbisignCircleFill=
'rbxassetid://114815486361826',
chineseyuanrenminbisignGaugeChartLefthalfRighthalf='rbxassetid://97094906069731'
,chineseyuanrenminbisignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://126731737755120',chineseyuanrenminbisignRing=
'rbxassetid://112854552761514',chineseyuanrenminbisignRingDashed=
'rbxassetid://115412544764832',chineseyuanrenminbisignSquare=
'rbxassetid://106069486060345',chineseyuanrenminbisignSquareFill=
'rbxassetid://138122887058133',circle='rbxassetid://77351764572440',
circleAndLineHorizontal='rbxassetid://122925249738775',
circleAndLineHorizontalFill='rbxassetid://71198922364962',circleBadgeCheckmark=
'rbxassetid://76550446292600',circleBadgeCheckmarkFill=
'rbxassetid://122095089484720',circleBadgeExclamationmark=
'rbxassetid://118464207416019',circleBadgeExclamationmarkFill=
'rbxassetid://116397404821243',circleBadgeMinus='rbxassetid://140477954283913',
circleBadgeMinusFill='rbxassetid://112082222631971',circleBadgePlus=
'rbxassetid://74158974124425',circleBadgePlusFill='rbxassetid://102359232710725'
,circleBadgeQuestionmark='rbxassetid://84105237816535',
circleBadgeQuestionmarkFill='rbxassetid://107508739526231',circleBadgeXmark=
'rbxassetid://75715075079912',circleBadgeXmarkFill=
'rbxassetid://108800417260586',circleBottomhalfFilled=
'rbxassetid://93772228869204',circleBottomhalfFilledInverse=
'rbxassetid://100348037143199',circleBottomrighthalfPatternCheckered=
'rbxassetid://94163682140627',circleCircle='rbxassetid://71758569358351',
circleCircleFill='rbxassetid://110715985557449',circleDashed=
'rbxassetid://80873602883859',circleDashedRectangle=
'rbxassetid://128384956155268',circleDotted='rbxassetid://105302007261428',
circleDottedAndCircle='rbxassetid://88524317015989',circleDottedCircle=
'rbxassetid://73982217657417',circleDottedCircleFill=
'rbxassetid://74317409458538',circleFill='rbxassetid://132609101842833',
circleFilledIpad='rbxassetid://118410279026483',circleFilledIpadFill=
'rbxassetid://138916749945804',circleFilledIpadLandscape=
'rbxassetid://98083380583955',circleFilledIpadLandscapeFill=
'rbxassetid://107790137388850',circleFilledIphone='rbxassetid://122754969849891'
,circleFilledIphoneFill='rbxassetid://97935781680008',
circleFilledPatternDiagonallineRectangle='rbxassetid://84692213314115',
circleGrid2x1='rbxassetid://126879867009619',circleGrid2x1Fill=
'rbxassetid://118105257533357',circleGrid2x1LeftFilled=
'rbxassetid://107581688387909',circleGrid2x1RightFilled=
'rbxassetid://75046917797428',circleGrid2x2='rbxassetid://136579717851920',
circleGrid2x2Fill='rbxassetid://114454902590504',
circleGrid2x2TopleftCheckmarkFilled='rbxassetid://89913344340254',circleGrid3x3=
'rbxassetid://94448556408587',circleGrid3x3Circle='rbxassetid://132776725985146'
,circleGrid3x3CircleFill='rbxassetid://81436799538589',circleGrid3x3Fill=
'rbxassetid://76516952302413',circleGridCross='rbxassetid://111852820274501',
circleGridCrossDownFilled='rbxassetid://122925459913152',circleGridCrossFill=
'rbxassetid://110500814372682',circleGridCrossLeftFilled=
'rbxassetid://90411348277462',circleGridCrossRightFilled=
'rbxassetid://73198600977610',circleGridCrossUpFilled=
'rbxassetid://140639075369170',circleHexagongrid='rbxassetid://115766942150344',
circleHexagongridCircle='rbxassetid://134748651158488',
circleHexagongridCircleFill='rbxassetid://81549201853724',circleHexagongridFill=
'rbxassetid://110857449976245',circleHexagonpath='rbxassetid://85916325663168',
circleHexagonpathFill='rbxassetid://137583129970644',circleLefthalfFilled=
'rbxassetid://97825242263995',circleLefthalfFilledInverse=
'rbxassetid://95093722032121',circleLefthalfFilledRighthalfStripedHorizontal=
'rbxassetid://108512481868469',
circleLefthalfFilledRighthalfStripedHorizontalInverse=
'rbxassetid://82874739567085',circleLefthalfStripedHorizontal=
'rbxassetid://93826488584783',circleLefthalfStripedHorizontalInverse=
'rbxassetid://97829908032507',circleOnSquare='rbxassetid://97779341422775',
circleOnSquareIntersectionDotted='rbxassetid://120659491389976',
circleOnSquareMerge='rbxassetid://78931485816076',circleRectangleDashed=
'rbxassetid://125046628567305',circleRectangleFilledPatternDiagonalline=
'rbxassetid://113655167746204',circleRighthalfFilled=
'rbxassetid://78313004126214',circleRighthalfFilledInverse=
'rbxassetid://114210082181728',circleSlash='rbxassetid://116513396479523',
circleSlashFill='rbxassetid://72982692904185',circleSquare=
'rbxassetid://122783494268200',circleSquareFill='rbxassetid://102931934264046',
circleTophalfFilled='rbxassetid://91615555048147',circleTophalfFilledInverse=
'rbxassetid://140511657377882',circlebadge='rbxassetid://91455021015370',
circlebadge2='rbxassetid://125133188090635',circlebadge2Fill=
'rbxassetid://96114399074311',circlebadgeFill='rbxassetid://100596250735760',
clear='rbxassetid://86389300199089',clearFill='rbxassetid://94709581536945',
clipboard='rbxassetid://131839244113738',clipboardFill=
'rbxassetid://134384663588746',clock='rbxassetid://118494205518216',
clockArrowTrianglehead2CounterclockwiseRotate90='rbxassetid://115137084749223',
clockArrowTriangleheadClockwiseRotate90PathDotted='rbxassetid://109195459563384'
,clockArrowTriangleheadCounterclockwiseRotate90='rbxassetid://89226626342302',
clockBadge='rbxassetid://115327553111420',clockBadgeAirplane=
'rbxassetid://88667530307852',clockBadgeAirplaneFill=
'rbxassetid://91355511171537',clockBadgeCheckmark='rbxassetid://83498643149203',
clockBadgeCheckmarkFill='rbxassetid://77828663867935',clockBadgeExclamationmark=
'rbxassetid://130712128950744',clockBadgeExclamationmarkFill=
'rbxassetid://120350431132914',clockBadgeFill='rbxassetid://88269624942481',
clockBadgeQuestionmark='rbxassetid://140419785571575',clockBadgeQuestionmarkFill
='rbxassetid://89682323416952',clockBadgeXmark='rbxassetid://100973182473925',
clockBadgeXmarkFill='rbxassetid://81230821710210',clockCircle=
'rbxassetid://91059286420704',clockCircleFill='rbxassetid://85551007246698',
clockFill='rbxassetid://111548973303929',cloud='rbxassetid://105957371924111',
cloudBolt='rbxassetid://97438995257422',cloudBoltCircle=
'rbxassetid://102467209270679',cloudBoltCircleFill='rbxassetid://86605548042299'
,cloudBoltFill='rbxassetid://108857264198967',cloudBoltRain=
'rbxassetid://119315902598803',cloudBoltRainCircle='rbxassetid://75667970292186'
,cloudBoltRainCircleFill='rbxassetid://101185113049432',cloudBoltRainFill=
'rbxassetid://105734599370391',cloudCircle='rbxassetid://75595371373901',
cloudCircleFill='rbxassetid://100351258374804',cloudDrizzle=
'rbxassetid://118772556442825',cloudDrizzleCircle='rbxassetid://134748373557774'
,cloudDrizzleCircleFill='rbxassetid://116611455603766',cloudDrizzleFill=
'rbxassetid://132048424719835',cloudFill='rbxassetid://81929130099582',cloudFog=
'rbxassetid://88316772996583',cloudFogCircle='rbxassetid://91971244695380',
cloudFogCircleFill='rbxassetid://107932107751895',cloudFogFill=
'rbxassetid://140333418982777',cloudHail='rbxassetid://109477306347948',
cloudHailCircle='rbxassetid://111935689657678',cloudHailCircleFill=
'rbxassetid://113958303878705',cloudHailFill='rbxassetid://88337312532907',
cloudHeavyrain='rbxassetid://119601076239239',cloudHeavyrainCircle=
'rbxassetid://113148606737260',cloudHeavyrainCircleFill=
'rbxassetid://93841246166430',cloudHeavyrainFill='rbxassetid://88892490794016',
cloudMoon='rbxassetid://81142197017787',cloudMoonBolt=
'rbxassetid://122762071192157',cloudMoonBoltCircle=
'rbxassetid://130186762815605',cloudMoonBoltCircleFill=
'rbxassetid://88717135754929',cloudMoonBoltFill='rbxassetid://132632709705945',
cloudMoonCircle='rbxassetid://101603611950498',cloudMoonCircleFill=
'rbxassetid://74403315229993',cloudMoonFill='rbxassetid://131415283779316',
cloudMoonRain='rbxassetid://123752971704324',cloudMoonRainCircle=
'rbxassetid://91321181570006',cloudMoonRainCircleFill=
'rbxassetid://99999475568772',cloudMoonRainFill='rbxassetid://87465253394705',
cloudRain='rbxassetid://73594633942444',cloudRainCircle=
'rbxassetid://135073289493994',cloudRainCircleFill=
'rbxassetid://107231577649535',cloudRainFill='rbxassetid://117683267149042',
cloudRainbowCrop='rbxassetid://100908348397951',cloudRainbowCropFill=
'rbxassetid://126657444384304',cloudSleet='rbxassetid://91110633048392',
cloudSleetCircle='rbxassetid://96915941810109',cloudSleetCircleFill=
'rbxassetid://126874750937247',cloudSleetFill='rbxassetid://92708514394653',
cloudSnow='rbxassetid://105037978809526',cloudSnowCircle=
'rbxassetid://133719082885022',cloudSnowCircleFill='rbxassetid://87293225773295'
,cloudSnowFill='rbxassetid://86264528698010',cloudSun=
'rbxassetid://90643281481332',cloudSunBolt='rbxassetid://131455198927801',
cloudSunBoltCircle='rbxassetid://125648631050001',cloudSunBoltCircleFill=
'rbxassetid://84471773945729',cloudSunBoltFill='rbxassetid://112333664152593',
cloudSunCircle='rbxassetid://113784163787232',cloudSunCircleFill=
'rbxassetid://129881246784143',cloudSunFill='rbxassetid://124208622832507',
cloudSunRain='rbxassetid://85585305376134',cloudSunRainCircle=
'rbxassetid://101680887245230',cloudSunRainCircleFill=
'rbxassetid://112862634540035',cloudSunRainFill='rbxassetid://96381809768088',
coat='rbxassetid://138683384493955',coatCircle='rbxassetid://92296510927085',
coatCircleFill='rbxassetid://96955835035080',coatFill=
'rbxassetid://85343931320953',coloncurrencysign='rbxassetid://70610561043896',
coloncurrencysignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://121730405897063',coloncurrencysignBankBuilding=
'rbxassetid://70532040854333',coloncurrencysignBankBuildingFill=
'rbxassetid://112013535327197',coloncurrencysignCircle=
'rbxassetid://72472930693295',coloncurrencysignCircleFill=
'rbxassetid://87581274361487',coloncurrencysignGaugeChartLefthalfRighthalf=
'rbxassetid://81410792250924',
coloncurrencysignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://117861383134809',coloncurrencysignRing=
'rbxassetid://90425639365292',coloncurrencysignRingDashed=
'rbxassetid://123033006789245',coloncurrencysignSquare=
'rbxassetid://123318002368880',coloncurrencysignSquareFill=
'rbxassetid://92584369957351',comb='rbxassetid://91258665510440',combFill=
'rbxassetid://75935642339195',command='rbxassetid://106412276667065',
commandCircle='rbxassetid://132133934704997',commandCircleFill=
'rbxassetid://129804286574492',commandSquare='rbxassetid://125800470592169',
commandSquareFill='rbxassetid://80987808644627',compassDrawing=
'rbxassetid://132752181112407',computermouse='rbxassetid://74192781750262',
computermouseFill='rbxassetid://122971754731281',cone=
'rbxassetid://136677139752159',coneFill='rbxassetid://102731753185841',
contactSensor='rbxassetid://78390045190635',contactSensorFill=
'rbxassetid://107941147884609',contextualmenuAndPointerArrow=
'rbxassetid://104765402938366',control='rbxassetid://122628517345884',
convertibleSide='rbxassetid://88056497032702',convertibleSideAirCirculate=
'rbxassetid://98143091298529',convertibleSideAirCirculateFill=
'rbxassetid://100789655329813',convertibleSideAirFresh=
'rbxassetid://133740317868475',convertibleSideAirFreshFill=
'rbxassetid://102875203407099',convertibleSideAndExclamationmark=
'rbxassetid://98977481869672',convertibleSideAndExclamationmarkFill=
'rbxassetid://75447287801799',convertibleSideArrowLeftAndRight=
'rbxassetid://86781979323180',convertibleSideArrowLeftAndRightFill=
'rbxassetid://132788293962828',convertibleSideArrowTriangleheadBackward=
'rbxassetid://78827119413410',convertibleSideArrowTriangleheadBackwardFill=
'rbxassetid://104495302305788',convertibleSideArrowTriangleheadForward=
'rbxassetid://74390894600361',convertibleSideArrowTriangleheadForwardAndBackward
='rbxassetid://111786845642207',
convertibleSideArrowTriangleheadForwardAndBackwardFill=
'rbxassetid://72089424692194',convertibleSideArrowTriangleheadForwardFill=
'rbxassetid://78071911288530',convertibleSideArrowtriangleDown=
'rbxassetid://94542124218390',convertibleSideArrowtriangleDownFill=
'rbxassetid://106589035606078',convertibleSideArrowtriangleUp=
'rbxassetid://120865137899154',convertibleSideArrowtriangleUpArrowtriangleDown=
'rbxassetid://74101665628817',
convertibleSideArrowtriangleUpArrowtriangleDownFill=
'rbxassetid://71804342394234',convertibleSideArrowtriangleUpFill=
'rbxassetid://110118762963000',convertibleSideFill=
'rbxassetid://109733797871544',convertibleSideFrontOpen=
'rbxassetid://131524948510161',convertibleSideFrontOpenCrop=
'rbxassetid://85705555161852',convertibleSideFrontOpenCropFill=
'rbxassetid://114271147567717',convertibleSideFrontOpenFill=
'rbxassetid://95366775755410',convertibleSideHillDescentControl=
'rbxassetid://71720992446425',convertibleSideHillDescentControlFill=
'rbxassetid://88324401650008',convertibleSideHillDown=
'rbxassetid://108529567267858',
convertibleSideHillDownAndGaugeOpenWithLinesNeedle25percentAndArrowtriangle=
'rbxassetid://107975199810052',
convertibleSideHillDownAndGaugeOpenWithLinesNeedle25percentAndArrowtriangleFill=
'rbxassetid://125979092076687',convertibleSideHillDownFill=
'rbxassetid://131556252117231',convertibleSideHillUp=
'rbxassetid://105616874495927',convertibleSideHillUpFill=
'rbxassetid://129311678978143',convertibleSideLock=
'rbxassetid://112823065019915',convertibleSideLockFill=
'rbxassetid://111356282511740',convertibleSideLockOpen=
'rbxassetid://113482904283039',convertibleSideLockOpenFill=
'rbxassetid://84761446199699',cooktop='rbxassetid://88834174813644',cooktopFill=
'rbxassetid://129568808107438',cpu='rbxassetid://124308556421451',cpuFill=
'rbxassetid://103202408856526',creditcard='rbxassetid://81319932685854',
creditcardAndNumbers='rbxassetid://94218302612103',
creditcardArrowTrianglehead2ClockwiseRotate90='rbxassetid://137775649489006',
creditcardCircle='rbxassetid://115901734894378',creditcardCircleFill=
'rbxassetid://94325401969577',creditcardFill='rbxassetid://137205665020340',
creditcardRewards='rbxassetid://105224490601254',creditcardRewardsFill=
'rbxassetid://122388666096808',creditcardTrianglebadgeExclamationmark=
'rbxassetid://128465389472705',creditcardTrianglebadgeExclamationmarkFill=
'rbxassetid://134046944234972',creditcardViewfinder=
'rbxassetid://107524109192728',cricketBall='rbxassetid://107557445941992',
cricketBallCircle='rbxassetid://108046247330406',cricketBallCircleFill=
'rbxassetid://97707187254281',cricketBallFill='rbxassetid://124235035918902',
crop='rbxassetid://76267866840424',cropRotate='rbxassetid://112208131294712',
cross='rbxassetid://117291897053132',crossCase='rbxassetid://85203105573889',
crossCaseCircle='rbxassetid://80349542022495',crossCaseCircleFill=
'rbxassetid://98787370493226',crossCaseFill='rbxassetid://108835778654525',
crossCircle='rbxassetid://127430494755623',crossCircleFill=
'rbxassetid://74611741813087',crossFill='rbxassetid://108668325552210',crossVial
='rbxassetid://139445714175616',crossVialFill='rbxassetid://134692391930699',
crown='rbxassetid://75313755369933',crownFill='rbxassetid://113744176715187',
cruzeirosign='rbxassetid://126565022512426',
cruzeirosignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://74744481288811',cruzeirosignBankBuilding=
'rbxassetid://131787371672007',cruzeirosignBankBuildingFill=
'rbxassetid://124556542732831',cruzeirosignCircle='rbxassetid://108994326873163'
,cruzeirosignCircleFill='rbxassetid://92977831088874',
cruzeirosignGaugeChartLefthalfRighthalf='rbxassetid://112870188799383',
cruzeirosignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://82432008607525',
cruzeirosignRing='rbxassetid://73494393849183',cruzeirosignRingDashed=
'rbxassetid://122245608823787',cruzeirosignSquare='rbxassetid://109516455287703'
,cruzeirosignSquareFill='rbxassetid://104979541987178',cube=
'rbxassetid://99085671007116',cubeCircle='rbxassetid://74137474397517',
cubeCircleFill='rbxassetid://134571955348253',cubeFill=
'rbxassetid://98135374759567',cubeTransparent='rbxassetid://80565642502539',
cubeTransparentFill='rbxassetid://126536927378991',cupAndHeatWaves=
'rbxassetid://74078659549582',cupAndHeatWavesFill='rbxassetid://131989311539655'
,cupAndSaucer='rbxassetid://72514825273993',cupAndSaucerFill=
'rbxassetid://119230927536360',curlybraces='rbxassetid://129788587428692',
curlybracesSquare='rbxassetid://129045503496668',curlybracesSquareFill=
'rbxassetid://123432376294099',curtainsClosed='rbxassetid://101807311791533',
curtainsOpen='rbxassetid://122412153002106',cylinder=
'rbxassetid://128980035962696',cylinderFill='rbxassetid://111577778638579',
cylinderSplit1x2='rbxassetid://127863382272636',cylinderSplit1x2Fill=
'rbxassetid://88248682215663',dCircle='rbxassetid://139348136095117',dCircleFill
='rbxassetid://119613399981436',dSquare='rbxassetid://117814926531634',
dSquareFill='rbxassetid://108548965881707',danishkronesign=
'rbxassetid://131739695626393',
danishkronesignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://105778149797087',danishkronesignBankBuilding=
'rbxassetid://107331257039732',danishkronesignBankBuildingFill=
'rbxassetid://110654132210408',danishkronesignCircle=
'rbxassetid://71261610464877',danishkronesignCircleFill=
'rbxassetid://81494386563215',danishkronesignGaugeChartLefthalfRighthalf=
'rbxassetid://96925630529917',
danishkronesignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://137030980787422',danishkronesignRing='rbxassetid://97079964616730'
,danishkronesignRingDashed='rbxassetid://133795816793944',danishkronesignSquare=
'rbxassetid://124245881324486',danishkronesignSquareFill=
'rbxassetid://85244239694374',decreaseIndent='rbxassetid://88589482580566',
decreaseQuotelevel='rbxassetid://116467773015005',degreesignCelsius=
'rbxassetid://89989256393732',degreesignFahrenheit=
'rbxassetid://104819176215784',dehumidifier='rbxassetid://124560404977624',
dehumidifierFill='rbxassetid://137890774655891',deleteBackward=
'rbxassetid://97287995124465',deleteBackwardFill='rbxassetid://91714832072413',
deleteForward='rbxassetid://126070993467794',deleteForwardFill=
'rbxassetid://101546743976234',deleteLeft='rbxassetid://129252293455681',
deleteLeftFill='rbxassetid://103030116283513',deleteRight=
'rbxassetid://107553771079065',deleteRightFill='rbxassetid://111969523580666',
deskclock='rbxassetid://128981521415034',deskclockFill=
'rbxassetid://77381200322602',desktopcomputer='rbxassetid://107766434710685',
desktopcomputerAndArrowDown='rbxassetid://72903302027184',
desktopcomputerAndMacbook='rbxassetid://123097897733245',
desktopcomputerBadgeCheckmark='rbxassetid://113146178110887',
desktopcomputerBadgeShieldCheckmark='rbxassetid://92532490405861',
desktopcomputerTrianglebadgeExclamationmark='rbxassetid://105558126089739',
deskview='rbxassetid://121089828258182',deskviewFill=
'rbxassetid://117267790741411',dialHigh='rbxassetid://73577823641217',
dialHighFill='rbxassetid://138768311720985',dialLow=
'rbxassetid://73103006519174',dialLowFill='rbxassetid://98791719425867',
dialMedium='rbxassetid://125486906119866',dialMediumFill=
'rbxassetid://101803104687767',diamond='rbxassetid://100452260150000',
diamondBottomhalfFilled='rbxassetid://119790712303249',diamondCircle=
'rbxassetid://85000945258354',diamondCircleFill='rbxassetid://130795247628751',
diamondFill='rbxassetid://135326379832516',diamondLefthalfFilled=
'rbxassetid://111934789670996',diamondRighthalfFilled=
'rbxassetid://116111158801385',diamondTophalfFilled=
'rbxassetid://110438206140750',dice='rbxassetid://101607867208664',diceFill=
'rbxassetid://132513568893953',dieFace1='rbxassetid://93511055000612',
dieFace1Fill='rbxassetid://101810661456778',dieFace2=
'rbxassetid://97436037441224',dieFace2Fill='rbxassetid://107358011011984',
dieFace3='rbxassetid://120095408336817',dieFace3Fill=
'rbxassetid://72211886706012',dieFace4='rbxassetid://106899046151642',
dieFace4Fill='rbxassetid://84516247319545',dieFace5=
'rbxassetid://129586843517471',dieFace5Fill='rbxassetid://89575782614538',
dieFace6='rbxassetid://118434046380828',dieFace6Fill=
'rbxassetid://79919588843970',digitalcrownArrowClockwise=
'rbxassetid://113038599465284',digitalcrownArrowClockwiseFill=
'rbxassetid://125537117338217',digitalcrownArrowCounterclockwise=
'rbxassetid://91415160437248',digitalcrownArrowCounterclockwiseFill=
'rbxassetid://135521665801021',digitalcrownHorizontalArrowClockwise=
'rbxassetid://103189101099622',digitalcrownHorizontalArrowClockwiseFill=
'rbxassetid://134814847872732',digitalcrownHorizontalArrowCounterclockwise=
'rbxassetid://127977551895798',digitalcrownHorizontalArrowCounterclockwiseFill=
'rbxassetid://128234037577616',digitalcrownHorizontalPress=
'rbxassetid://96420371236309',digitalcrownHorizontalPressFill=
'rbxassetid://128765548994947',digitalcrownPress='rbxassetid://78417970263455',
digitalcrownPressFill='rbxassetid://117434984032927',directcurrent=
'rbxassetid://131799886298156',dishwasher='rbxassetid://87710012658252',
dishwasherCircle='rbxassetid://101196794042889',dishwasherCircleFill=
'rbxassetid://125490773916925',dishwasherFill='rbxassetid://107996524055597',
display='rbxassetid://129576202104478',display2='rbxassetid://104707340982471',
displayAndArrowDown='rbxassetid://125200972320706',displayAndScrewdriver=
'rbxassetid://88947373615358',displayTrianglebadgeExclamationmark=
'rbxassetid://74215695749246',distributeHorizontal='rbxassetid://70445662962312'
,distributeHorizontalCenter='rbxassetid://80437477134490',
distributeHorizontalCenterFill='rbxassetid://119697341124280',
distributeHorizontalFill='rbxassetid://81547792767797',distributeHorizontalLeft=
'rbxassetid://107554802550012',distributeHorizontalLeftFill=
'rbxassetid://94354038132668',distributeHorizontalRight=
'rbxassetid://108003162032073',distributeHorizontalRightFill=
'rbxassetid://73286830298055',distributeVertical='rbxassetid://76030980380639',
distributeVerticalBottom='rbxassetid://125371761357477',
distributeVerticalBottomFill='rbxassetid://101717338101326',
distributeVerticalCenter='rbxassetid://118073308636361',
distributeVerticalCenterFill='rbxassetid://87414672077654',
distributeVerticalFill='rbxassetid://107674189896422',distributeVerticalTop=
'rbxassetid://76620811418507',distributeVerticalTopFill=
'rbxassetid://124081343166798',divide='rbxassetid://79215535863886',divideCircle
='rbxassetid://113145969264461',divideCircleFill='rbxassetid://123714081526266',
divideSquare='rbxassetid://111006580979570',divideSquareFill=
'rbxassetid://105584343535953',dockArrowDownRectangle=
'rbxassetid://79678007655929',dockArrowUpRectangle=
'rbxassetid://103809621231446',dockRectangle='rbxassetid://116915015149373',
document='rbxassetid://124796793806383',documentBadgeArrowUp=
'rbxassetid://123268508542979',documentBadgeArrowUpFill=
'rbxassetid://90983453010822',documentBadgeClock='rbxassetid://110094088290911',
documentBadgeClockFill='rbxassetid://125542031580318',documentBadgeEllipsis=
'rbxassetid://117158658954997',documentBadgeEllipsisFill=
'rbxassetid://76114071497838',documentBadgeGearshape=
'rbxassetid://77928287555730',documentBadgeGearshapeFill=
'rbxassetid://84274129384253',documentBadgePlus='rbxassetid://131301015581855',
documentBadgePlusFill='rbxassetid://119897017745527',documentCircle=
'rbxassetid://115432124967770',documentCircleFill='rbxassetid://81065392469526',
documentFill='rbxassetid://102002749838822',documentOnClipboard=
'rbxassetid://93839365739605',documentOnClipboardFill=
'rbxassetid://112586343230192',documentOnDocument='rbxassetid://107508916580138'
,documentOnDocumentFill='rbxassetid://74882215448372',documentOnTrash=
'rbxassetid://121400057226892',documentOnTrashFill='rbxassetid://81536752681320'
,documentViewfinder='rbxassetid://121616818189343',documentViewfinderFill=
'rbxassetid://70819609607946',dog='rbxassetid://117451448671654',dogCircle=
'rbxassetid://105450428935772',dogCircleFill='rbxassetid://98543830678397',
dogFill='rbxassetid://92378974436965',dollarsign='rbxassetid://114666918383783',
dollarsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://89635002789945',dollarsignBankBuilding=
'rbxassetid://105920028179036',dollarsignBankBuildingFill=
'rbxassetid://132899638309304',dollarsignCircle='rbxassetid://102492024082780',
dollarsignCircleFill='rbxassetid://71830017801030',
dollarsignGaugeChartLefthalfRighthalf='rbxassetid://129052029424233',
dollarsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://78922584980415',
dollarsignRing='rbxassetid://135226063978248',dollarsignRingDashed=
'rbxassetid://112067357041185',dollarsignSquare='rbxassetid://137446308450569',
dollarsignSquareFill='rbxassetid://135191385118976',dongsign=
'rbxassetid://131470155313598',dongsignArrowTriangleheadCounterclockwiseRotate90
='rbxassetid://78266004720742',dongsignBankBuilding=
'rbxassetid://93291038171841',dongsignBankBuildingFill=
'rbxassetid://134028059678599',dongsignCircle='rbxassetid://78578408861421',
dongsignCircleFill='rbxassetid://97359784120063',
dongsignGaugeChartLefthalfRighthalf='rbxassetid://126111988758130',
dongsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://132144167952188',
dongsignRing='rbxassetid://77819742511029',dongsignRingDashed=
'rbxassetid://74495453626839',dongsignSquare='rbxassetid://101387193682428',
dongsignSquareFill='rbxassetid://82247973365677',doorFrenchClosed=
'rbxassetid://72674657682992',doorFrenchOpen='rbxassetid://71650861494809',
doorGarageClosed='rbxassetid://76882327011413',
doorGarageClosedTrianglebadgeExclamationmark='rbxassetid://77158083946409',
doorGarageDoubleBayClosed='rbxassetid://80981083592931',
doorGarageDoubleBayClosedTrianglebadgeExclamationmark=
'rbxassetid://126845743919696',doorGarageDoubleBayOpen=
'rbxassetid://80126036062871',
doorGarageDoubleBayOpenTrianglebadgeExclamationmark=
'rbxassetid://81451333563049',doorGarageOpen='rbxassetid://119088880497339',
doorGarageOpenTrianglebadgeExclamationmark='rbxassetid://87728457848837',
doorLeftHandClosed='rbxassetid://135128964730567',doorLeftHandOpen=
'rbxassetid://83765067910138',doorRightHandClosed='rbxassetid://110182995960620'
,doorRightHandOpen='rbxassetid://106043050975592',doorSlidingLeftHandClosed=
'rbxassetid://87632050696273',doorSlidingLeftHandOpen=
'rbxassetid://78005731822470',doorSlidingRightHandClosed=
'rbxassetid://128301725703574',doorSlidingRightHandOpen=
'rbxassetid://84574799071754',dotArrowtrianglesUpRightDownLeftCircle=
'rbxassetid://131760626023852',dotCarTopRadiowaves2RearLeftRearRearRight=
'rbxassetid://90311472253622',dotCarTopRadiowaves2RearLeftRearRearRightFill=
'rbxassetid://108359832311792',dotCircleAndHandPointUpLeftFill=
'rbxassetid://128518318617359',dotCircleAndPointerArrow=
'rbxassetid://111417453409808',dotCircleViewfinder='rbxassetid://96091854795879'
,dotCrosshair='rbxassetid://111411956958702',dotRadiowavesForward=
'rbxassetid://107117879701250',dotRadiowavesLeftAndRight=
'rbxassetid://130013546237645',dotRadiowavesRight='rbxassetid://83521858010518',
dotRadiowavesUpForward='rbxassetid://137844140535297',dotScope=
'rbxassetid://121005306065203',dotScopeDisplay='rbxassetid://133201781049322',
dotScopeLaptopcomputer='rbxassetid://90429518888381',dotSquare=
'rbxassetid://136076872200498',dotSquareFill='rbxassetid://73545757794889',
dotSquareshape='rbxassetid://115421844852656',dotSquareshapeFill=
'rbxassetid://131818751234447',dotSquareshapeSplit2x2=
'rbxassetid://111473506170578',dotViewfinder='rbxassetid://113016896284675',
dotsAndLineVerticalAndPointerArrowRectangle='rbxassetid://92115593886595',dpad=
'rbxassetid://112675157052076',dpadDownFilled='rbxassetid://112396763830267',
dpadFill='rbxassetid://122140108936845',dpadLeftFilled=
'rbxassetid://87311733672381',dpadRightFilled='rbxassetid://130745043343873',
dpadUpFilled='rbxassetid://134846020479603',drone='rbxassetid://78690133487507',
droneFill='rbxassetid://133031915402113',drop='rbxassetid://118377862469616',
dropCircle='rbxassetid://71240363811196',dropCircleFill=
'rbxassetid://88945998378182',dropDegreesign='rbxassetid://77649680596146',
dropDegreesignFill='rbxassetid://94362397679873',dropDegreesignSlash=
'rbxassetid://110482944076581',dropDegreesignSlashFill=
'rbxassetid://71669882733811',dropFill='rbxassetid://85349169340444',
dropHalffull='rbxassetid://91240077152622',dropKeypadRectangle=
'rbxassetid://70840237003373',dropKeypadRectangleFill=
'rbxassetid://125931309084435',dropTransmission='rbxassetid://101407191451186',
dropTriangle='rbxassetid://84183738522099',dropTriangleFill=
'rbxassetid://90805606987344',dryer='rbxassetid://104968399783199',dryerCircle=
'rbxassetid://120074568345505',dryerCircleFill='rbxassetid://111654035272821',
dryerFill='rbxassetid://136500520544490',duffleBag='rbxassetid://88923927348266'
,duffleBagFill='rbxassetid://99592794377634',dumbbell=
'rbxassetid://131658449144206',dumbbellFill='rbxassetid://102647184300293',
eCircle='rbxassetid://103742511836669',eCircleFill='rbxassetid://95162271305994'
,eSquare='rbxassetid://101495905268164',eSquareFill=
'rbxassetid://90805151324988',ear='rbxassetid://93167066666811',
earBadgeCheckmark='rbxassetid://118287233995960',earBadgeWaveform=
'rbxassetid://96770578331307',earFill='rbxassetid://74082991593421',
earTrianglebadgeExclamationmark='rbxassetid://77722678508145',earbudLeft=
'rbxassetid://90739084119105',earbudRight='rbxassetid://70630407521257',earbuds=
'rbxassetid://79401980005894',earbudsBoneConduction=
'rbxassetid://102719667776865',earbudsBoneConductionLeft=
'rbxassetid://86650415805297',earbudsBoneConductionRight=
'rbxassetid://87823184053006',earbudsCase='rbxassetid://140476224814409',
earbudsCaseFill='rbxassetid://111148501625727',earbudsInEar=
'rbxassetid://109063443614950',earbudsInEarLeft='rbxassetid://106492595834880',
earbudsInEarRight='rbxassetid://125584398701209',earbudsStemless=
'rbxassetid://99846906714452',earbudsStemlessLeft='rbxassetid://98848664729808',
earbudsStemlessRight='rbxassetid://122917678453047',earpods=
'rbxassetid://74278614743942',eject='rbxassetid://120765145931209',ejectCircle=
'rbxassetid://104332244648721',ejectCircleFill='rbxassetid://140628083571758',
ejectFill='rbxassetid://71327024184670',electronicTollCollection=
'rbxassetid://136913224306634',electronicTollCollectionRectangle=
'rbxassetid://128593575467422',electronicTollCollectionRectangleFill=
'rbxassetid://112309550756861',electronicTollCollectionRectangleSlash=
'rbxassetid://95728463970850',electronicTollCollectionRectangleSlashFill=
'rbxassetid://125346120200754',
electronicTollCollectionRectangleTrianglebadgeExclamationmark=
'rbxassetid://90467983733711',
electronicTollCollectionRectangleTrianglebadgeExclamationmarkFill=
'rbxassetid://94360087978127',ellipsis='rbxassetid://124361135229418',
ellipsisBubble='rbxassetid://126798458237130',ellipsisBubbleFill=
'rbxassetid://79069091642283',ellipsisCalendar='rbxassetid://124805713057261',
ellipsisCircle='rbxassetid://113784442485781',ellipsisCircleBadge=
'rbxassetid://130244717421714',ellipsisCircleBadgeFill=
'rbxassetid://82159336176596',ellipsisCircleFill='rbxassetid://122334603788097',
ellipsisCurlybraces='rbxassetid://107977531225704',ellipsisMessage=
'rbxassetid://80558243872489',ellipsisMessageFill='rbxassetid://132149801435799'
,ellipsisRectangle='rbxassetid://83388291144122',ellipsisRectangleFill=
'rbxassetid://117142929617215',ellipsisVerticalBubble=
'rbxassetid://72015532400199',ellipsisVerticalBubbleFill=
'rbxassetid://116203081148848',ellipsisViewfinder='rbxassetid://94637317080093',
engineCombustion='rbxassetid://92302125521416',
engineCombustionBadgeExclamationmark='rbxassetid://76712122282370',
engineCombustionBadgeExclamationmarkFill='rbxassetid://89069786141963',
engineCombustionFill='rbxassetid://99487338050460',
engineEmissionAndDrop2WaterWaveBelow='rbxassetid://87305393152974',
engineEmissionAndExclamationmark='rbxassetid://114464443552928',
engineEmissionAndFilter='rbxassetid://127179695323664',entryLeverKeypad=
'rbxassetid://118027558598911',entryLeverKeypadFill=
'rbxassetid://118392320417223',entryLeverKeypadTrianglebadgeExclamationmark=
'rbxassetid://137214809742459',entryLeverKeypadTrianglebadgeExclamationmarkFill=
'rbxassetid://75303037354493',envelope='rbxassetid://106830949234316',
envelopeAndArrow3Down='rbxassetid://117149942819834',envelopeAndArrow3DownFill=
'rbxassetid://79378848825247',envelopeAndArrowTriangleheadBranch=
'rbxassetid://104523224983991',envelopeAndArrowTriangleheadBranchFill=
'rbxassetid://100707359290921',envelopeAndHandRaised=
'rbxassetid://78356406355168',envelopeAndHandRaisedFill=
'rbxassetid://107728689664001',envelopeBadge='rbxassetid://93576600755016',
envelopeBadgeFill='rbxassetid://116876097672908',envelopeBadgeMinus=
'rbxassetid://130814870797193',envelopeBadgeMinusFill=
'rbxassetid://76642074014671',envelopeBadgePersonCrop=
'rbxassetid://105705646416572',envelopeBadgePersonCropFill=
'rbxassetid://79242615918290',envelopeBadgePlus='rbxassetid://107588810121905',
envelopeBadgePlusFill='rbxassetid://117435240335216',
envelopeBadgeShieldHalfFilled='rbxassetid://140515835641833',
envelopeBadgeShieldHalfFilledFill='rbxassetid://81922163345893',envelopeCircle=
'rbxassetid://136873814874607',envelopeCircleFill='rbxassetid://111051395244412'
,envelopeFill='rbxassetid://127507072697652',envelopeFront=
'rbxassetid://122506649095626',envelopeFrontFill='rbxassetid://104299711974123',
envelopeOpen='rbxassetid://100877378812997',envelopeOpenBadgeClock=
'rbxassetid://100228997184187',envelopeOpenBadgeClockFill=
'rbxassetid://71379109503673',envelopeOpenFill='rbxassetid://85763605331589',
envelopeStack='rbxassetid://87145831106000',envelopeStackFill=
'rbxassetid://123354851119926',environments='rbxassetid://70776273028500',
environmentsCircle='rbxassetid://105521855459846',environmentsCircleFill=
'rbxassetid://104935991380195',environmentsFill='rbxassetid://105632666776458',
environmentsSlash='rbxassetid://104765532616958',environmentsSlashCircle=
'rbxassetid://78007865985283',environmentsSlashCircleFill=
'rbxassetid://91748220983526',environmentsSlashFill=
'rbxassetid://75145189219140',equal='rbxassetid://81928889048192',equalCircle=
'rbxassetid://79382669951914',equalCircleFill='rbxassetid://86371079165197',
equalSquare='rbxassetid://79936726576289',equalSquareFill=
'rbxassetid://104388387477692',eraser='rbxassetid://131139520079799',
eraserBadgeXmark='rbxassetid://80645300777635',eraserBadgeXmarkFill=
'rbxassetid://110922477071984',eraserFill='rbxassetid://108705331076102',
eraserLineDashed='rbxassetid://76929643809652',eraserLineDashedFill=
'rbxassetid://81891295746096',eraserSlash='rbxassetid://136706221907454',
eraserSlashFill='rbxassetid://83380933600841',eraserTrianglebadgeExclamationmark
='rbxassetid://105319193923389',eraserTrianglebadgeExclamationmarkFill=
'rbxassetid://98230351518742',escape='rbxassetid://133789976668661',esim=
'rbxassetid://131303958903026',esimFill='rbxassetid://86636914917717',eurosign=
'rbxassetid://82925976421520',eurosignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://99526735026701',eurosignBankBuilding='rbxassetid://89483927128263'
,eurosignBankBuildingFill='rbxassetid://108012299359814',eurosignCircle=
'rbxassetid://140251138055967',eurosignCircleFill='rbxassetid://87679842728207',
eurosignGaugeChartLefthalfRighthalf='rbxassetid://114065360584117',
eurosignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://72338154797145',
eurosignRing='rbxassetid://134135280297519',eurosignRingDashed=
'rbxassetid://81105789808808',eurosignSquare='rbxassetid://118857707505659',
eurosignSquareFill='rbxassetid://85458107728944',eurozonesign=
'rbxassetid://106111186723394',
eurozonesignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://96687530526973',eurozonesignBankBuilding=
'rbxassetid://101771619031147',eurozonesignBankBuildingFill=
'rbxassetid://125592888577896',eurozonesignCircle='rbxassetid://114038615288773'
,eurozonesignCircleFill='rbxassetid://82301050010699',
eurozonesignGaugeChartLefthalfRighthalf='rbxassetid://123642748750595',
eurozonesignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://139979957807793'
,eurozonesignRing='rbxassetid://105011696428140',eurozonesignRingDashed=
'rbxassetid://93030366645152',eurozonesignSquare='rbxassetid://105016814402384',
eurozonesignSquareFill='rbxassetid://112888895324413',evCharger=
'rbxassetid://97939002186339',evChargerArrowtriangleLeft=
'rbxassetid://78531181074444',evChargerArrowtriangleLeftFill=
'rbxassetid://129265466116295',evChargerArrowtriangleRight=
'rbxassetid://73571516368733',evChargerArrowtriangleRightFill=
'rbxassetid://133332267515902',evChargerExclamationmark=
'rbxassetid://71284303991317',evChargerExclamationmarkFill=
'rbxassetid://73242768019910',evChargerFill='rbxassetid://135357327350082',
evChargerSlash='rbxassetid://119392987220859',evChargerSlashFill=
'rbxassetid://117895186907340',evPlugAcGbT='rbxassetid://124662740249269',
evPlugAcGbTFill='rbxassetid://97796850278979',evPlugAcType1=
'rbxassetid://105942058683839',evPlugAcType1Fill='rbxassetid://87565275880095',
evPlugAcType2='rbxassetid://109491714154043',evPlugAcType2Fill=
'rbxassetid://136749111819862',evPlugDcCcs1='rbxassetid://111896860617304',
evPlugDcCcs1Fill='rbxassetid://135258359505207',evPlugDcCcs2=
'rbxassetid://89215178998105',evPlugDcCcs2Fill='rbxassetid://132543470987109',
evPlugDcChademo='rbxassetid://118191687282802',evPlugDcChademoFill=
'rbxassetid://115432509818358',evPlugDcGbT='rbxassetid://99656496942844',
evPlugDcGbTFill='rbxassetid://76353846804034',evPlugDcNacs=
'rbxassetid://128552750734766',evPlugDcNacsFill='rbxassetid://72083694831744',
exclamationmark='rbxassetid://105489810797515',exclamationmark2=
'rbxassetid://138940800083904',exclamationmark3='rbxassetid://96062819103435',
exclamationmarkApplewatch='rbxassetid://70859643783699',
exclamationmarkArrowTrianglehead2ClockwiseRotate90='rbxassetid://87427147276965'
,exclamationmarkArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://120412511808787',exclamationmarkBrakesignal=
'rbxassetid://73594089484983',exclamationmarkBubble=
'rbxassetid://95883853598192',exclamationmarkBubbleCircle=
'rbxassetid://75160551591562',exclamationmarkBubbleCircleFill=
'rbxassetid://94137052992947',exclamationmarkBubbleFill=
'rbxassetid://120138139594154',exclamationmarkCircle=
'rbxassetid://124980468238909',exclamationmarkCircleFill=
'rbxassetid://118467060972001',exclamationmarkIcloud=
'rbxassetid://76695078178669',exclamationmarkIcloudFill=
'rbxassetid://131208143896029',exclamationmarkLock=
'rbxassetid://106384248542364',exclamationmarkLockFill=
'rbxassetid://109195138504207',exclamationmarkMagnifyingglass=
'rbxassetid://92574625679787',exclamationmarkMessage=
'rbxassetid://138562668016011',exclamationmarkMessageFill=
'rbxassetid://132448718389247',exclamationmarkOctagon=
'rbxassetid://137747340644933',exclamationmarkOctagonFill=
'rbxassetid://105984442012480',exclamationmarkQuestionmark=
'rbxassetid://92436083739221',exclamationmarkShield=
'rbxassetid://112276285190661',exclamationmarkShieldFill=
'rbxassetid://83928233244528',exclamationmarkSquare=
'rbxassetid://80560347785472',exclamationmarkSquareFill=
'rbxassetid://116001111235657',exclamationmarkTirepressure=
'rbxassetid://91238920104209',exclamationmarkTransmission=
'rbxassetid://102604704591626',exclamationmarkTriangle=
'rbxassetid://107822175160368',exclamationmarkTriangleFill=
'rbxassetid://114874558386316',exclamationmarkTriangleTextPage=
'rbxassetid://129462348942674',exclamationmarkTriangleTextPageFill=
'rbxassetid://87819250469566',exclamationmarkWarninglight=
'rbxassetid://129488082492608',exclamationmarkWarninglightFill=
'rbxassetid://109563951728634',externaldrive='rbxassetid://80232260950078',
externaldriveBadgeCheckmark='rbxassetid://127808980665357',
externaldriveBadgeExclamationmark='rbxassetid://138816851149625',
externaldriveBadgeIcloud='rbxassetid://134043240118451',externaldriveBadgeMinus=
'rbxassetid://93718911516002',externaldriveBadgePersonCrop=
'rbxassetid://131468993878865',externaldriveBadgePlus=
'rbxassetid://120630286835644',externaldriveBadgeQuestionmark=
'rbxassetid://95288528165880',externaldriveBadgeTimemachine=
'rbxassetid://98241082446758',externaldriveBadgeWifi=
'rbxassetid://79557098906550',externaldriveBadgeXmark=
'rbxassetid://122842097395468',externaldriveConnectedToLineBelow=
'rbxassetid://130509526753365',externaldriveConnectedToLineBelowFill=
'rbxassetid://95945834927517',externaldriveFill='rbxassetid://111188851017397',
externaldriveFillBadgeCheckmark='rbxassetid://122664727689795',
externaldriveFillBadgeExclamationmark='rbxassetid://139443980658229',
externaldriveFillBadgeIcloud='rbxassetid://111658236910597',
externaldriveFillBadgeMinus='rbxassetid://138322201910424',
externaldriveFillBadgePersonCrop='rbxassetid://128003216586005',
externaldriveFillBadgePlus='rbxassetid://114554892368455',
externaldriveFillBadgeQuestionmark='rbxassetid://76317585194672',
externaldriveFillBadgeTimemachine='rbxassetid://106899749894347',
externaldriveFillBadgeWifi='rbxassetid://72486197994673',
externaldriveFillBadgeXmark='rbxassetid://98595452028940',
externaldriveFillTrianglebadgeExclamationmark='rbxassetid://98123683908795',
externaldriveTrianglebadgeExclamationmark='rbxassetid://106750675144343',eye=
'rbxassetid://111055543166389',eyeCircle='rbxassetid://107190923208298',
eyeCircleFill='rbxassetid://135793946948356',eyeFill=
'rbxassetid://77235800413545',eyeHalfClosed='rbxassetid://77207415123898',
eyeHalfClosedFill='rbxassetid://113576230437344',eyeSlash=
'rbxassetid://119498809185323',eyeSlashCircle='rbxassetid://132524732539324',
eyeSlashCircleFill='rbxassetid://103154106288950',eyeSlashFill=
'rbxassetid://81778913526360',eyeSquare='rbxassetid://95242157997285',
eyeSquareFill='rbxassetid://126083471525251',eyeTrianglebadgeExclamationmark=
'rbxassetid://89778592723539',eyeTrianglebadgeExclamationmarkFill=
'rbxassetid://109952601896975',eyebrow='rbxassetid://138874889430672',eyedropper
='rbxassetid://122887134386475',eyedropperFull='rbxassetid://91873383286058',
eyedropperHalffull='rbxassetid://110456203480587',eyeglasses=
'rbxassetid://81029046265930',eyeglassesSlash='rbxassetid://124054712844326',
eyes='rbxassetid://78678714479511',eyesInverse='rbxassetid://79358575110860',
fCircle='rbxassetid://120350062281724',fCircleFill='rbxassetid://85941153359585'
,fCursive='rbxassetid://121112615529535',fCursiveCircle=
'rbxassetid://118145351560598',fCursiveCircleFill='rbxassetid://79867894564651',
fCursiveSlash='rbxassetid://82830783756850',fSquare=
'rbxassetid://138319993082756',fSquareFill='rbxassetid://103704824525018',
faceDashed='rbxassetid://115965004314399',faceDashedFill=
'rbxassetid://85477783339034',faceSmiling='rbxassetid://78636671887809',
faceSmilingInverse='rbxassetid://89152469821007',faceid=
'rbxassetid://95081493927842',facemask='rbxassetid://99230089481142',
facemaskFill='rbxassetid://87358950271058',fan='rbxassetid://104236974088032',
fanAndLightCeiling='rbxassetid://107305354245185',fanAndLightCeilingFill=
'rbxassetid://79601447011266',fanBadgeArrowUpAndDownAndArrowLeftAndRight=
'rbxassetid://113898399220980',fanBadgeArrowUpAndDownAndArrowLeftAndRightFill=
'rbxassetid://124338705997821',fanBadgeAutomatic='rbxassetid://134431646851799',
fanBadgeAutomaticFill='rbxassetid://136476275577129',fanCeiling=
'rbxassetid://96665114463151',fanCeilingFill='rbxassetid://138987560662626',
fanCircle='rbxassetid://70854430102154',fanCircleFill=
'rbxassetid://117605684696409',fanDesk='rbxassetid://114862891749128',
fanDeskFill='rbxassetid://93071498661239',fanFill='rbxassetid://77445753113040',
fanFloor='rbxassetid://109401757372728',fanFloorFill=
'rbxassetid://81956707111904',fanGaugeOpen='rbxassetid://116949823717075',
fanOscillation='rbxassetid://140040205782084',fanOscillationFill=
'rbxassetid://95940480162856',fanSlash='rbxassetid://88988508555905',
fanSlashFill='rbxassetid://91132405529736',faxmachine=
'rbxassetid://98463604203755',faxmachineFill='rbxassetid://133950895418533',
ferry='rbxassetid://104552475367467',ferryFill='rbxassetid://135078258733005',
fibrechannel='rbxassetid://98030247410028',fieldOfViewUltrawide=
'rbxassetid://114057852168752',fieldOfViewUltrawideFill=
'rbxassetid://83751448891679',fieldOfViewWide='rbxassetid://96004890847207',
fieldOfViewWideFill='rbxassetid://137899142791405',figure=
'rbxassetid://110213647176253',figure2='rbxassetid://138808308230792',
figure2AndChildHoldinghands='rbxassetid://102771612569539',figure2ArmsOpen=
'rbxassetid://92288487915619',figure2Circle='rbxassetid://94206481778029',
figure2CircleFill='rbxassetid://129196015079328',figure2LeftHoldinghands=
'rbxassetid://100537110943074',figure2RightHoldinghands=
'rbxassetid://119526196408433',figureAmericanFootball=
'rbxassetid://91093899560194',figureAmericanFootballCircle=
'rbxassetid://73526407862195',figureAmericanFootballCircleFill=
'rbxassetid://125643881949592',figureAndChildHoldinghands=
'rbxassetid://98234143469303',figureArchery='rbxassetid://91146536671456',
figureArcheryCircle='rbxassetid://126114292998662',figureArcheryCircleFill=
'rbxassetid://139817967546701',figureArmsOpen='rbxassetid://91747516641055',
figureAustralianFootball='rbxassetid://95801007298244',
figureAustralianFootballCircle='rbxassetid://114857191814495',
figureAustralianFootballCircleFill='rbxassetid://96243863542381',figureBadminton
='rbxassetid://76476251455465',figureBadmintonCircle=
'rbxassetid://89195992414478',figureBadmintonCircleFill=
'rbxassetid://114444756914155',figureBarre='rbxassetid://110544483608061',
figureBarreCircle='rbxassetid://135837012134445',figureBarreCircleFill=
'rbxassetid://127607577096807',figureBaseball='rbxassetid://97849278832097',
figureBaseballCircle='rbxassetid://114304222609888',figureBaseballCircleFill=
'rbxassetid://123393585149734',figureBasketball='rbxassetid://70458961636702',
figureBasketballCircle='rbxassetid://133108362517016',figureBasketballCircleFill
='rbxassetid://118516949832825',figureBowling='rbxassetid://87430224954145',
figureBowlingCircle='rbxassetid://92291199433832',figureBowlingCircleFill=
'rbxassetid://83424282699101',figureBoxing='rbxassetid://85849315745014',
figureBoxingCircle='rbxassetid://108865178770018',figureBoxingCircleFill=
'rbxassetid://103918369499522',figureChild='rbxassetid://110064985871757',
figureChildAndLock='rbxassetid://73535299980245',figureChildAndLockFill=
'rbxassetid://71218198833330',figureChildAndLockOpen=
'rbxassetid://108694793096350',figureChildAndLockOpenFill=
'rbxassetid://125673450678214',figureChildCircle='rbxassetid://81423849672320',
figureChildCircleFill='rbxassetid://82354760379505',figureClimbing=
'rbxassetid://120678773528166',figureClimbingCircle=
'rbxassetid://120120038724485',figureClimbingCircleFill=
'rbxassetid://132240539735275',figureCooldown='rbxassetid://92932298775664',
figureCooldownCircle='rbxassetid://116114093190620',figureCooldownCircleFill=
'rbxassetid://121936218292480',figureCoreTraining='rbxassetid://123052378717278'
,figureCoreTrainingCircle='rbxassetid://72613470087026',
figureCoreTrainingCircleFill='rbxassetid://89955491381206',figureCricket=
'rbxassetid://116071257091954',figureCricketCircle=
'rbxassetid://134233377377044',figureCricketCircleFill=
'rbxassetid://72325772863796',figureCrossTraining='rbxassetid://73418643387339',
figureCrossTrainingCircle='rbxassetid://72456662052046',
figureCrossTrainingCircleFill='rbxassetid://77850653388101',figureCurling=
'rbxassetid://133903499695545',figureCurlingCircle=
'rbxassetid://125787714674084',figureCurlingCircleFill=
'rbxassetid://119803288536473',figureDance='rbxassetid://82823113185452',
figureDanceCircle='rbxassetid://129448865281588',figureDanceCircleFill=
'rbxassetid://99987979019469',figureDiscSports='rbxassetid://97666785998608',
figureDiscSportsCircle='rbxassetid://111401373560932',figureDiscSportsCircleFill
='rbxassetid://71691339998044',figureElliptical='rbxassetid://103011240387444',
figureEllipticalCircle='rbxassetid://131478855174617',figureEllipticalCircleFill
='rbxassetid://104064244810242',figureEquestrianSports=
'rbxassetid://113004809635168',figureEquestrianSportsCircle=
'rbxassetid://85892477745919',figureEquestrianSportsCircleFill=
'rbxassetid://97852470945461',figureFall='rbxassetid://78917096397322',
figureFallCircle='rbxassetid://120817219815068',figureFallCircleFill=
'rbxassetid://131207735766677',figureFencing='rbxassetid://88727868093192',
figureFencingCircle='rbxassetid://106032601544162',figureFencingCircleFill=
'rbxassetid://98217461416689',figureFieldHockey='rbxassetid://97574076033706',
figureFieldHockeyCircle='rbxassetid://104931510950353',
figureFieldHockeyCircleFill='rbxassetid://130141092753763',figureFishing=
'rbxassetid://97928619322397',figureFishingCircle='rbxassetid://110360794736175'
,figureFishingCircleFill='rbxassetid://136506795917678',figureFlexibility=
'rbxassetid://102643174092557',figureFlexibilityCircle=
'rbxassetid://107915680316772',figureFlexibilityCircleFill=
'rbxassetid://131562203565834',figureGolf='rbxassetid://108030447882352',
figureGolfCircle='rbxassetid://75259568650212',figureGolfCircleFill=
'rbxassetid://78581076173636',figureGymnastics='rbxassetid://110639559835087',
figureGymnasticsCircle='rbxassetid://123858181191697',figureGymnasticsCircleFill
='rbxassetid://102190187170175',figureHandCycling='rbxassetid://105321743988781'
,figureHandCyclingCircle='rbxassetid://85007285093415',
figureHandCyclingCircleFill='rbxassetid://91530596716487',figureHandball=
'rbxassetid://115838442370626',figureHandballCircle=
'rbxassetid://70759727291307',figureHandballCircleFill=
'rbxassetid://119475164895904',figureHighintensityIntervaltraining=
'rbxassetid://76737326147568',figureHighintensityIntervaltrainingCircle=
'rbxassetid://80397488456920',figureHighintensityIntervaltrainingCircleFill=
'rbxassetid://98555009338585',figureHiking='rbxassetid://95596775753518',
figureHikingCircle='rbxassetid://138221954074963',figureHikingCircleFill=
'rbxassetid://88401031990874',figureHockey='rbxassetid://90256454862003',
figureHockeyCircle='rbxassetid://94383755687550',figureHockeyCircleFill=
'rbxassetid://136334583328074',figureHunting='rbxassetid://104331371922960',
figureHuntingCircle='rbxassetid://127527976370774',figureHuntingCircleFill=
'rbxassetid://81475211852623',figureIceHockey='rbxassetid://107583026067878',
figureIceHockeyCircle='rbxassetid://118193809208332',figureIceHockeyCircleFill=
'rbxassetid://111522178102920',figureIceSkating='rbxassetid://135617966638935',
figureIceSkatingCircle='rbxassetid://138214165419825',figureIceSkatingCircleFill
='rbxassetid://98770013409291',figureIndoorCycle='rbxassetid://80969946389201',
figureIndoorCycleCircle='rbxassetid://134390336862454',
figureIndoorCycleCircleFill='rbxassetid://95479995212663',figureIndoorRowing=
'rbxassetid://84240034900060',figureIndoorRowingCircle=
'rbxassetid://136778375743254',figureIndoorRowingCircleFill=
'rbxassetid://106705352180779',figureIndoorSoccer='rbxassetid://101995448808644'
,figureIndoorSoccerCircle='rbxassetid://98693757644611',
figureIndoorSoccerCircleFill='rbxassetid://115853013474405',figureJumprope=
'rbxassetid://112314762824266',figureJumpropeCircle=
'rbxassetid://131867434437690',figureJumpropeCircleFill=
'rbxassetid://82475486201404',figureKickboxing='rbxassetid://71547925836213',
figureKickboxingCircle='rbxassetid://111712574527276',figureKickboxingCircleFill
='rbxassetid://90368178358769',figureLacrosse='rbxassetid://131555411154516',
figureLacrosseCircle='rbxassetid://125836289315347',figureLacrosseCircleFill=
'rbxassetid://128431465303733',figureMartialArts='rbxassetid://135745150496906',
figureMartialArtsCircle='rbxassetid://119796908146049',
figureMartialArtsCircleFill='rbxassetid://113266429878233',figureMindAndBody=
'rbxassetid://103860527144324',figureMindAndBodyCircle=
'rbxassetid://106858535249934',figureMindAndBodyCircleFill=
'rbxassetid://138656680070798',figureMixedCardio='rbxassetid://121353989624376',
figureMixedCardioCircle='rbxassetid://124865593330962',
figureMixedCardioCircleFill='rbxassetid://133316587934777',figureOpenWaterSwim=
'rbxassetid://99134075301751',figureOpenWaterSwimCircle=
'rbxassetid://135738254961736',figureOpenWaterSwimCircleFill=
'rbxassetid://73573960201738',figureOutdoorCycle='rbxassetid://94389585238104',
figureOutdoorCycleCircle='rbxassetid://104565216263530',
figureOutdoorCycleCircleFill='rbxassetid://85407648912798',figureOutdoorRowing=
'rbxassetid://97214365802337',figureOutdoorRowingCircle=
'rbxassetid://110435954031642',figureOutdoorRowingCircleFill=
'rbxassetid://89699495932779',figureOutdoorSoccer='rbxassetid://111514292464663'
,figureOutdoorSoccerCircle='rbxassetid://78804307665587',
figureOutdoorSoccerCircleFill='rbxassetid://84815226618665',figurePickleball=
'rbxassetid://103908880748936',figurePickleballCircle=
'rbxassetid://136344983007459',figurePickleballCircleFill=
'rbxassetid://71182848325604',figurePilates='rbxassetid://108752699363037',
figurePilatesCircle='rbxassetid://113467772255977',figurePilatesCircleFill=
'rbxassetid://132440163880650',figurePlay='rbxassetid://125930862891634',
figurePlayCircle='rbxassetid://77583381787661',figurePlayCircleFill=
'rbxassetid://134400678202309',figurePoolSwim='rbxassetid://137689299313028',
figurePoolSwimCircle='rbxassetid://89666987644291',figurePoolSwimCircleFill=
'rbxassetid://92630998913564',figureRacquetball='rbxassetid://99889129274922',
figureRacquetballCircle='rbxassetid://129504301099369',
figureRacquetballCircleFill='rbxassetid://78638439110000',figureRoll=
'rbxassetid://92890687621570',figureRollCircle='rbxassetid://115447965292337',
figureRollCircleFill='rbxassetid://78758776592473',figureRollRunningpace=
'rbxassetid://103258084151079',figureRollRunningpaceCircle=
'rbxassetid://119394454598856',figureRollRunningpaceCircleFill=
'rbxassetid://71338146987517',figureRolling='rbxassetid://96544717929655',
figureRollingCircle='rbxassetid://130243270009292',figureRollingCircleFill=
'rbxassetid://81436799482432',figureRugby='rbxassetid://109148647244317',
figureRugbyCircle='rbxassetid://121768904436252',figureRugbyCircleFill=
'rbxassetid://124286693797479',figureRun='rbxassetid://113159658934048',
figureRunCircle='rbxassetid://107121944951745',figureRunCircleFill=
'rbxassetid://134197067094502',figureRunSquareStack=
'rbxassetid://112190947223332',figureRunSquareStackFill=
'rbxassetid://79407822771023',figureRunTreadmill='rbxassetid://72845166434800',
figureRunTreadmillCircle='rbxassetid://100935270576310',
figureRunTreadmillCircleFill='rbxassetid://90094962041285',figureSailing=
'rbxassetid://101372266533215',figureSailingCircle=
'rbxassetid://132108643163002',figureSailingCircleFill=
'rbxassetid://105305283071460',figureSeatedSeatbelt=
'rbxassetid://106552740060057',figureSeatedSeatbeltAndAirbagOff=
'rbxassetid://98186865777515',figureSeatedSeatbeltAndAirbagOn=
'rbxassetid://91150009110532',figureSeatedSeatbeltLeftDriveSeats1=
'rbxassetid://85946255939735',figureSeatedSeatbeltLeftDriveSeats11=
'rbxassetid://79678492525730',figureSeatedSeatbeltLeftDriveSeats11Fill=
'rbxassetid://129089651219076',figureSeatedSeatbeltLeftDriveSeats12=
'rbxassetid://93143466530275',figureSeatedSeatbeltLeftDriveSeats12Fill=
'rbxassetid://105065766784936',figureSeatedSeatbeltLeftDriveSeats1Fill=
'rbxassetid://113481718311782',figureSeatedSeatbeltLeftDriveSeats2=
'rbxassetid://75226954452926',figureSeatedSeatbeltLeftDriveSeats22=
'rbxassetid://101101160267265',figureSeatedSeatbeltLeftDriveSeats222=
'rbxassetid://118909297450613',figureSeatedSeatbeltLeftDriveSeats222Fill=
'rbxassetid://93791159425519',figureSeatedSeatbeltLeftDriveSeats223=
'rbxassetid://78036184824175',figureSeatedSeatbeltLeftDriveSeats223Fill=
'rbxassetid://92266074429811',figureSeatedSeatbeltLeftDriveSeats22Fill=
'rbxassetid://116221944283747',figureSeatedSeatbeltLeftDriveSeats23=
'rbxassetid://127024965881932',figureSeatedSeatbeltLeftDriveSeats232=
'rbxassetid://117513359005767',figureSeatedSeatbeltLeftDriveSeats232Fill=
'rbxassetid://82664717592655',figureSeatedSeatbeltLeftDriveSeats233=
'rbxassetid://119483437512021',figureSeatedSeatbeltLeftDriveSeats233Fill=
'rbxassetid://71345890100600',figureSeatedSeatbeltLeftDriveSeats23Fill=
'rbxassetid://131875209072674',figureSeatedSeatbeltLeftDriveSeats2Fill=
'rbxassetid://79587393298221',figureSeatedSeatbeltLeftDriveSeats3=
'rbxassetid://74148888025573',figureSeatedSeatbeltLeftDriveSeats33=
'rbxassetid://116237917433348',figureSeatedSeatbeltLeftDriveSeats333=
'rbxassetid://74481906475225',figureSeatedSeatbeltLeftDriveSeats333Fill=
'rbxassetid://117037471107630',figureSeatedSeatbeltLeftDriveSeats33Fill=
'rbxassetid://128363081385683',figureSeatedSeatbeltLeftDriveSeats3Fill=
'rbxassetid://107990648154861',figureSeatedSideLeft=
'rbxassetid://77481740371688',figureSeatedSideLeftAirDistributionIndirect=
'rbxassetid://79336558528852',figureSeatedSideLeftAirDistributionLower=
'rbxassetid://127392790646826',
figureSeatedSideLeftAirDistributionLowerAngledAndUpperAngled=
'rbxassetid://112078481062843',figureSeatedSideLeftAirDistributionMiddle=
'rbxassetid://78706747938686',figureSeatedSideLeftAirDistributionMiddleAndLower=
'rbxassetid://100831410587505',
figureSeatedSideLeftAirDistributionMiddleAndLowerAngled=
'rbxassetid://75259277174467',figureSeatedSideLeftAirDistributionUpper=
'rbxassetid://74202315829466',
figureSeatedSideLeftAirDistributionUpperAndMiddleAndLower=
'rbxassetid://114743873533519',
figureSeatedSideLeftAirDistributionUpperAngledAndDottedlineAndLowerAngled=
'rbxassetid://74199874956849',
figureSeatedSideLeftAirDistributionUpperAngledAndLowerAngled=
'rbxassetid://105111573378012',
figureSeatedSideLeftAirDistributionUpperAngledAndMiddle=
'rbxassetid://89346514926957',
figureSeatedSideLeftAirDistributionUpperAngledAndMiddleAndLowerAngled=
'rbxassetid://80317618153794',figureSeatedSideLeftAirbagOff=
'rbxassetid://86363377016551',figureSeatedSideLeftAirbagOff2=
'rbxassetid://131286776968522',figureSeatedSideLeftAirbagOn=
'rbxassetid://82060834562607',figureSeatedSideLeftAirbagOn2=
'rbxassetid://93600189126601',figureSeatedSideLeftAutomatic=
'rbxassetid://96448990318081',figureSeatedSideLeftFan=
'rbxassetid://95773073671580',figureSeatedSideLeftSteeringwheel=
'rbxassetid://99756030999052',figureSeatedSideLeftWindshieldFrontAndHeatWaves=
'rbxassetid://92505789545181',
figureSeatedSideLeftWindshieldFrontAndHeatWavesAirDistributionLower=
'rbxassetid://137421564202284',
figureSeatedSideLeftWindshieldFrontAndHeatWavesAirDistributionMiddle=
'rbxassetid://81437547930986',
figureSeatedSideLeftWindshieldFrontAndHeatWavesAirDistributionMiddleAndLower=
'rbxassetid://136445257646948',
figureSeatedSideLeftWindshieldFrontAndHeatWavesAirDistributionUpper=
'rbxassetid://88397915417301',
figureSeatedSideLeftWindshieldFrontAndHeatWavesAirDistributionUpperAndLower=
'rbxassetid://92086059777005',
figureSeatedSideLeftWindshieldFrontAndHeatWavesAirDistributionUpperAndMiddle=
'rbxassetid://88932521630743',
figureSeatedSideLeftWindshieldFrontAndHeatWavesAirDistributionUpperAndMiddleAndLower
='rbxassetid://89902175830534',figureSeatedSideRight=
'rbxassetid://136726651600980',figureSeatedSideRightAirDistributionIndirect=
'rbxassetid://89934262166596',figureSeatedSideRightAirDistributionLower=
'rbxassetid://73300805424690',
figureSeatedSideRightAirDistributionLowerAngledAndUpperAngled=
'rbxassetid://112866271533668',figureSeatedSideRightAirDistributionMiddle=
'rbxassetid://112098509009862',
figureSeatedSideRightAirDistributionMiddleAndLower=
'rbxassetid://140706729304027',
figureSeatedSideRightAirDistributionMiddleAndLowerAngled=
'rbxassetid://112352134374515',figureSeatedSideRightAirDistributionUpper=
'rbxassetid://98585139293028',
figureSeatedSideRightAirDistributionUpperAndMiddleAndLower=
'rbxassetid://129371148484159',
figureSeatedSideRightAirDistributionUpperAngledAndDottedlineAndLowerAngled=
'rbxassetid://71258161512054',
figureSeatedSideRightAirDistributionUpperAngledAndLowerAngled=
'rbxassetid://135103013327810',
figureSeatedSideRightAirDistributionUpperAngledAndMiddle=
'rbxassetid://103548517854131',
figureSeatedSideRightAirDistributionUpperAngledAndMiddleAndLowerAngled=
'rbxassetid://134493708141205',figureSeatedSideRightAirbagOff=
'rbxassetid://94058410986970',figureSeatedSideRightAirbagOff2=
'rbxassetid://100168980896563',figureSeatedSideRightAirbagOn=
'rbxassetid://116869659816584',figureSeatedSideRightAirbagOn2=
'rbxassetid://132818635170679',figureSeatedSideRightAutomatic=
'rbxassetid://111548991851424',figureSeatedSideRightChildLap=
'rbxassetid://76765306258341',figureSeatedSideRightFan=
'rbxassetid://105598745260535',figureSeatedSideRightSteeringwheel=
'rbxassetid://90687210460576',figureSeatedSideRightWindshieldFrontAndHeatWaves=
'rbxassetid://86526553476751',
figureSeatedSideRightWindshieldFrontAndHeatWavesAirDistributionLower=
'rbxassetid://81180167854979',
figureSeatedSideRightWindshieldFrontAndHeatWavesAirDistributionMiddle=
'rbxassetid://73749499001488',
figureSeatedSideRightWindshieldFrontAndHeatWavesAirDistributionMiddleAndLower=
'rbxassetid://97986729272402',
figureSeatedSideRightWindshieldFrontAndHeatWavesAirDistributionUpper=
'rbxassetid://86983951557733',
figureSeatedSideRightWindshieldFrontAndHeatWavesAirDistributionUpperAndLower=
'rbxassetid://83534989566232',
figureSeatedSideRightWindshieldFrontAndHeatWavesAirDistributionUpperAndMiddle=
'rbxassetid://110050631545966',
figureSeatedSideRightWindshieldFrontAndHeatWavesAirDistributionUpperAndMiddleAndLower
='rbxassetid://121091370522998',figureSkateboarding=
'rbxassetid://120531764531561',figureSkateboardingCircle=
'rbxassetid://89075859489979',figureSkateboardingCircleFill=
'rbxassetid://107428167276894',figureSkiingCrosscountry=
'rbxassetid://95376513080244',figureSkiingCrosscountryCircle=
'rbxassetid://91319383729536',figureSkiingCrosscountryCircleFill=
'rbxassetid://100781904758952',figureSkiingDownhill=
'rbxassetid://94948924926037',figureSkiingDownhillCircle=
'rbxassetid://124726796339746',figureSkiingDownhillCircleFill=
'rbxassetid://133873735630410',figureSnowboarding='rbxassetid://93552707410765',
figureSnowboardingCircle='rbxassetid://91124188336468',
figureSnowboardingCircleFill='rbxassetid://80006732027627',figureSocialdance=
'rbxassetid://98146311664249',figureSocialdanceCircle=
'rbxassetid://128341017230082',figureSocialdanceCircleFill=
'rbxassetid://90611956553302',figureSoftball='rbxassetid://121025439420310',
figureSoftballCircle='rbxassetid://102433510764523',figureSoftballCircleFill=
'rbxassetid://72322590245571',figureSquash='rbxassetid://115910960216751',
figureSquashCircle='rbxassetid://108255500001887',figureSquashCircleFill=
'rbxassetid://92599428723725',figureStairStepper='rbxassetid://97526748063558',
figureStairStepperCircle='rbxassetid://93435159397362',
figureStairStepperCircleFill='rbxassetid://86095743098312',figureStairs=
'rbxassetid://80928066724578',figureStairsCircle='rbxassetid://108007736984499',
figureStairsCircleFill='rbxassetid://97282222240704',figureStand=
'rbxassetid://94523389012151',figureStandDress='rbxassetid://81879939422801',
figureStandDressLineVerticalFigure='rbxassetid://75755749939997',
figureStandLineDottedFigureStand='rbxassetid://74698954943767',
figureStepTraining='rbxassetid://89082931649415',figureStepTrainingCircle=
'rbxassetid://83798887901170',figureStepTrainingCircleFill=
'rbxassetid://103288596282785',figureStrengthtrainingFunctional=
'rbxassetid://104636472308678',figureStrengthtrainingFunctionalCircle=
'rbxassetid://71887020003942',figureStrengthtrainingFunctionalCircleFill=
'rbxassetid://136782284825758',figureStrengthtrainingTraditional=
'rbxassetid://86384766229916',figureStrengthtrainingTraditionalCircle=
'rbxassetid://137707664026627',figureStrengthtrainingTraditionalCircleFill=
'rbxassetid://122169081610135',figureSurfing='rbxassetid://96005835479619',
figureSurfingCircle='rbxassetid://97406987955422',figureSurfingCircleFill=
'rbxassetid://71245429084383',figureTableTennis='rbxassetid://84629739397090',
figureTableTennisCircle='rbxassetid://75152334648876',
figureTableTennisCircleFill='rbxassetid://103942005472858',figureTaichi=
'rbxassetid://110478807239910',figureTaichiCircle='rbxassetid://75278727258055',
figureTaichiCircleFill='rbxassetid://71467374422500',figureTennis=
'rbxassetid://109986369675921',figureTennisCircle='rbxassetid://102155111733843'
,figureTennisCircleFill='rbxassetid://131743158972045',figureTrackAndField=
'rbxassetid://136538052321610',figureTrackAndFieldCircle=
'rbxassetid://103736888619277',figureTrackAndFieldCircleFill=
'rbxassetid://123902162029358',figureVolleyball='rbxassetid://94303634727000',
figureVolleyballCircle='rbxassetid://136059877417611',figureVolleyballCircleFill
='rbxassetid://79422865723683',figureWalk='rbxassetid://79511822122227',
figureWalkArrival='rbxassetid://126341672233316',figureWalkCircle=
'rbxassetid://103084762751916',figureWalkCircleFill=
'rbxassetid://91923043437287',figureWalkDeparture='rbxassetid://102119906082019'
,figureWalkDiamond='rbxassetid://114194884373216',figureWalkDiamondFill=
'rbxassetid://126938906614617',figureWalkMotion='rbxassetid://112030395820861',
figureWalkMotionTrianglebadgeExclamationmark='rbxassetid://106410785966772',
figureWalkSuitcaseRolling='rbxassetid://139035074665377',
figureWalkSuitcaseRollingCircle='rbxassetid://71150674505180',
figureWalkSuitcaseRollingCircleFill='rbxassetid://113530785990351',
figureWalkTreadmill='rbxassetid://73060375741248',figureWalkTreadmillCircle=
'rbxassetid://96470047881214',figureWalkTreadmillCircleFill=
'rbxassetid://98035075220974',figureWalkTriangle='rbxassetid://78276655377384',
figureWalkTriangleFill='rbxassetid://98344113716078',figureWaterFitness=
'rbxassetid://91138168805690',figureWaterFitnessCircle=
'rbxassetid://104634803350077',figureWaterFitnessCircleFill=
'rbxassetid://76002732032150',figureWaterpolo='rbxassetid://139456823152794',
figureWaterpoloCircle='rbxassetid://118729585287398',figureWaterpoloCircleFill=
'rbxassetid://83689434096148',figureWave='rbxassetid://126350733687264',
figureWaveCircle='rbxassetid://103869802030971',figureWaveCircleFill=
'rbxassetid://82754948484397',figureWrestling='rbxassetid://81893433411503',
figureWrestlingCircle='rbxassetid://134981808172623',figureWrestlingCircleFill=
'rbxassetid://94060023878868',figureYoga='rbxassetid://131429815541470',
figureYogaCircle='rbxassetid://98261961081745',figureYogaCircleFill=
'rbxassetid://110744567898624',filemenuAndPointerArrow=
'rbxassetid://80797128300493',filemenuAndSelection=
'rbxassetid://133250059618545',film='rbxassetid://86941949111966',filmCircle=
'rbxassetid://124831403308365',filmCircleFill='rbxassetid://110423444595475',
filmFill='rbxassetid://93532359201336',filmStack='rbxassetid://93437740666776',
filmStackFill='rbxassetid://77875956814737',finder=
'rbxassetid://105028817444613',fireExtinguisher='rbxassetid://115098785397510',
fireExtinguisherFill='rbxassetid://110280038568592',fireplace=
'rbxassetid://112954869899081',fireplaceFill='rbxassetid://75619406961928',
firewall='rbxassetid://126760488734970',firewallFill=
'rbxassetid://84638883483155',fireworks='rbxassetid://99494540765134',fish=
'rbxassetid://125194628802668',fishCircle='rbxassetid://124319918102703',
fishCircleFill='rbxassetid://77945311435195',fishFill=
'rbxassetid://106504216014923',flag='rbxassetid://122736846758534',flag2Crossed=
'rbxassetid://78844784013079',flag2CrossedCircle='rbxassetid://78584216590675',
flag2CrossedCircleFill='rbxassetid://86134708117980',flag2CrossedFill=
'rbxassetid://84296510463315',flagAndFlagFilledCrossed=
'rbxassetid://100745243139308',flagBadgeEllipsis='rbxassetid://92803573179376',
flagBadgeEllipsisFill='rbxassetid://91092814709500',flagCircle=
'rbxassetid://77170785894950',flagCircleFill='rbxassetid://99647307774250',
flagFill='rbxassetid://131669346188016',flagFilledAndFlagCrossed=
'rbxassetid://118037026885077',flagPatternCheckered=
'rbxassetid://125386743240339',flagPatternCheckered2Crossed=
'rbxassetid://115034252105148',flagPatternCheckeredCircle=
'rbxassetid://88125322330487',flagPatternCheckeredCircleFill=
'rbxassetid://80253578332366',flagPatternCheckeredLc=
'rbxassetid://79015272896327',flagSlash='rbxassetid://136165673088536',
flagSlashCircle='rbxassetid://92686158250177',flagSlashCircleFill=
'rbxassetid://111660599139888',flagSlashFill='rbxassetid://71634940997338',
flagSquare='rbxassetid://71479706467165',flagSquareFill=
'rbxassetid://100277208162099',flame='rbxassetid://108768547584054',flameCircle=
'rbxassetid://82747734852018',flameCircleFill='rbxassetid://86456375644145',
flameFill='rbxassetid://119270312720784',flameGaugeOpen=
'rbxassetid://113126271179041',flashlightOffCircle=
'rbxassetid://117830570878966',flashlightOffCircleFill=
'rbxassetid://72961865361048',flashlightOffFill='rbxassetid://101207421230626',
flashlightOnCircle='rbxassetid://79621682641826',flashlightOnCircleFill=
'rbxassetid://122824628666008',flashlightOnFill='rbxassetid://104818867409741',
flashlightSlash='rbxassetid://124642568340306',flashlightSlashCircle=
'rbxassetid://106284752783985',flashlightSlashCircleFill=
'rbxassetid://131469491577650',flask='rbxassetid://71705325441366',flaskFill=
'rbxassetid://73156360900679',fleuron='rbxassetid://84029089647656',fleuronFill=
'rbxassetid://132784812676419',flipphone='rbxassetid://86153595900991',
florinsign='rbxassetid://82815295829174',
florinsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://131747551393628',florinsignBankBuilding=
'rbxassetid://101348563612482',florinsignBankBuildingFill=
'rbxassetid://73257997241079',florinsignCircle='rbxassetid://102666413087615',
florinsignCircleFill='rbxassetid://101645622228381',
florinsignGaugeChartLefthalfRighthalf='rbxassetid://110044104730017',
florinsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://95345432829840',
florinsignRing='rbxassetid://140452194895022',florinsignRingDashed=
'rbxassetid://96409551878599',florinsignSquare='rbxassetid://131511638483506',
florinsignSquareFill='rbxassetid://99961037421189',flowchart=
'rbxassetid://112012575789082',flowchartFill='rbxassetid://85988923904543',
fluidBatteryblock='rbxassetid://87171297845261',fluidBrakesignal=
'rbxassetid://85409681443732',fluidCoolant='rbxassetid://94305313777913',
fluidTransmission='rbxassetid://81520433429253',fn=
'rbxassetid://136973223245238',folder='rbxassetid://125012160621787',
folderBadgeGearshape='rbxassetid://110354717501799',folderBadgeMinus=
'rbxassetid://105424398626891',folderBadgePersonCrop=
'rbxassetid://118262710565929',folderBadgePlus='rbxassetid://86162465085192',
folderBadgeQuestionmark='rbxassetid://96129876038348',folderCircle=
'rbxassetid://136623654746784',folderCircleFill='rbxassetid://71560799678637',
folderFill='rbxassetid://139360049720186',folderFillBadgeGearshape=
'rbxassetid://86874469728544',folderFillBadgeMinus=
'rbxassetid://126706256406221',folderFillBadgePersonCrop=
'rbxassetid://116825460620355',folderFillBadgePlus=
'rbxassetid://117713198610469',folderFillBadgeQuestionmark=
'rbxassetid://101253803162337',forkKnife='rbxassetid://88625690387037',
forkKnifeCircle='rbxassetid://114255039792324',forkKnifeCircleFill=
'rbxassetid://74351929405604',formfittingGamecontroller=
'rbxassetid://91206901253404',formfittingGamecontrollerFill=
'rbxassetid://88145883762633',forward='rbxassetid://113037057903118',
forwardCircle='rbxassetid://115172960663901',forwardCircleFill=
'rbxassetid://77543137210165',forwardEnd='rbxassetid://130719114360778',
forwardEndAlt='rbxassetid://119828878420743',forwardEndAltFill=
'rbxassetid://87004210011550',forwardEndCircle='rbxassetid://127570430828500',
forwardEndCircleFill='rbxassetid://77630986993530',forwardEndFill=
'rbxassetid://135891363464093',forwardFill='rbxassetid://118726763278360',
forwardFrame='rbxassetid://117425644293321',forwardFrameFill=
'rbxassetid://119062379885277',fossilShell='rbxassetid://108141425042235',
fossilShellFill='rbxassetid://92090095699092',francsign=
'rbxassetid://104329955978670',
francsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://123734401541585',francsignBankBuilding=
'rbxassetid://140379990360270',francsignBankBuildingFill=
'rbxassetid://101202870300556',francsignCircle='rbxassetid://92045068970438',
francsignCircleFill='rbxassetid://109010950306083',
francsignGaugeChartLefthalfRighthalf='rbxassetid://111277658129239',
francsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://82217136095508',
francsignRing='rbxassetid://124717081982767',francsignRingDashed=
'rbxassetid://111221536076635',francsignSquare='rbxassetid://117783360698586',
francsignSquareFill='rbxassetid://139671995243527',fryingPan=
'rbxassetid://95665944113167',fryingPanFill='rbxassetid://131190456038889',
fuelFilterWater='rbxassetid://114849224744442',fuelpump=
'rbxassetid://128432265591197',fuelpumpAndFilter='rbxassetid://132403377755564',
fuelpumpArrowtriangleLeft='rbxassetid://77219675885475',
fuelpumpArrowtriangleLeftFill='rbxassetid://105580200622844',
fuelpumpArrowtriangleRight='rbxassetid://78806432937161',
fuelpumpArrowtriangleRightFill='rbxassetid://116584326215382',fuelpumpCircle=
'rbxassetid://92321665086316',fuelpumpCircleFill='rbxassetid://109590002626519',
fuelpumpExclamationmark='rbxassetid://124935631685999',
fuelpumpExclamationmarkFill='rbxassetid://114211970014735',fuelpumpFill=
'rbxassetid://89804646715125',fuelpumpSlash='rbxassetid://112344106152981',
fuelpumpSlashFill='rbxassetid://131380848022445',fuelpumpThermometer=
'rbxassetid://105776255539239',fuelpumpThermometerFill=
'rbxassetid://91014703108179',['function']='rbxassetid://105988244015471',fx=
'rbxassetid://133676038076601',gCircle='rbxassetid://92465681841123',gCircleFill
='rbxassetid://117703861321064',gSquare='rbxassetid://98413842477799',
gSquareFill='rbxassetid://121882609445976',gamecontroller=
'rbxassetid://136899701592916',gamecontrollerCircle=
'rbxassetid://115838851199918',gamecontrollerCircleFill=
'rbxassetid://100513486519514',gamecontrollerFill='rbxassetid://107710782304881'
,gaugeChartLefthalfRighthalf='rbxassetid://88317515489831',
gaugeChartLeftthirdTopthirdRightthird='rbxassetid://117914290141041',gaugeOpen=
'rbxassetid://72661873101783',
gaugeOpenRighthalfDottedWithNeedleAndArrowTriangleheadBackward=
'rbxassetid://106683214692474',gaugeOpenWithLinesNeedle33percent=
'rbxassetid://96930938614147',
gaugeOpenWithLinesNeedle33percentAndArrowTriangleheadFrom0percentTo50percent=
'rbxassetid://72455258039917',gaugeOpenWithLinesNeedle33percentAndArrowtriangle=
'rbxassetid://129905955726763',gaugeOpenWithLinesNeedle67percentAndArrowtriangle
='rbxassetid://104277227320240',
gaugeOpenWithLinesNeedle67percentAndArrowtriangleAndCar=
'rbxassetid://74920052592412',gaugeOpenWithLinesNeedle84percentExclamation=
'rbxassetid://131118944109256',gaugeWithDotsNeedle0percent=
'rbxassetid://71374226674159',gaugeWithDotsNeedle100percent=
'rbxassetid://82175977935882',gaugeWithDotsNeedle33percent=
'rbxassetid://92757773478553',gaugeWithDotsNeedle50percent=
'rbxassetid://115384416941293',gaugeWithDotsNeedle67percent=
'rbxassetid://109864469246268',gaugeWithDotsNeedleBottom0percent=
'rbxassetid://74885486474683',gaugeWithDotsNeedleBottom100percent=
'rbxassetid://72579321556049',gaugeWithDotsNeedleBottom50percent=
'rbxassetid://120735235060336',gaugeWithDotsNeedleBottom50percentBadgeMinus=
'rbxassetid://140660728462393',gaugeWithDotsNeedleBottom50percentBadgePlus=
'rbxassetid://104491325572328',gaugeWithNeedle='rbxassetid://77994983474118',
gaugeWithNeedleFill='rbxassetid://72397110171923',gear=
'rbxassetid://133102912527371',gearBadge='rbxassetid://114821006676114',
gearBadgeCheckmark='rbxassetid://116948953549356',gearBadgeQuestionmark=
'rbxassetid://94190722082023',gearBadgeXmark='rbxassetid://102962629872900',
gearCircle='rbxassetid://80927635234925',gearCircleFill=
'rbxassetid://78683348854232',gearshape='rbxassetid://127735318557062',
gearshape2='rbxassetid://95807134365871',gearshape2Fill=
'rbxassetid://132207224797441',gearshapeArrowTrianglehead2ClockwiseRotate90=
'rbxassetid://130625453122583',gearshapeCircle='rbxassetid://70545341819845',
gearshapeCircleFill='rbxassetid://105088281654478',gearshapeFill=
'rbxassetid://98061574923339',gearshiftLayoutSixspeed=
'rbxassetid://83954102394855',gift='rbxassetid://130378804447034',giftCircle=
'rbxassetid://105693012395359',giftCircleFill='rbxassetid://75831198909976',
giftFill='rbxassetid://122811099954906',giftcard='rbxassetid://74001358275222',
giftcardFill='rbxassetid://80648895512474',globe='rbxassetid://75166161739358',
globeAmericas='rbxassetid://82208390184303',globeAmericasFill=
'rbxassetid://117216039216006',globeAsiaAustralia='rbxassetid://101174379892302'
,globeAsiaAustraliaFill='rbxassetid://125274596435685',globeBadgeChevronBackward
='rbxassetid://118399161412796',globeBadgeClock='rbxassetid://81246082209280',
globeBadgeClockFill='rbxassetid://133084199012794',globeCentralSouthAsia=
'rbxassetid://122671652893265',globeCentralSouthAsiaFill=
'rbxassetid://120361963069758',globeDesk='rbxassetid://74995477837587',
globeDeskFill='rbxassetid://98081463950992',globeEuropeAfrica=
'rbxassetid://136835539161615',globeEuropeAfricaFill=
'rbxassetid://130908776432341',globeFill='rbxassetid://108905553032102',glowplug
='rbxassetid://81022835723835',graduationcap='rbxassetid://125671553630065',
graduationcapCircle='rbxassetid://113915747930128',graduationcapCircleFill=
'rbxassetid://105306913757085',graduationcapFill='rbxassetid://133021220811177',
graph2d='rbxassetid://115562943064650',graph3d='rbxassetid://78081408845927',
greaterthan='rbxassetid://72942886858684',greaterthanCircle=
'rbxassetid://86205987134419',greaterthanCircleFill=
'rbxassetid://86221088035316',greaterthanSquare='rbxassetid://138820278327055',
greaterthanSquareFill='rbxassetid://82992552700740',greaterthanorequalto=
'rbxassetid://84817672830093',greaterthanorequaltoCircle=
'rbxassetid://129245462379395',greaterthanorequaltoCircleFill=
'rbxassetid://71490024938088',greaterthanorequaltoSquare=
'rbxassetid://74647437313735',greaterthanorequaltoSquareFill=
'rbxassetid://116032105238422',greetingcard='rbxassetid://71909891559012',
greetingcardFill='rbxassetid://129481097749734',grid=
'rbxassetid://137977554732401',gridCircle='rbxassetid://110916645524765',
gridCircleFill='rbxassetid://128975539852133',guaranisign=
'rbxassetid://112075506061752',
guaranisignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://126343329265137',guaranisignBankBuilding=
'rbxassetid://83180391553596',guaranisignBankBuildingFill=
'rbxassetid://75636756682974',guaranisignCircle='rbxassetid://85150493060456',
guaranisignCircleFill='rbxassetid://133828485193738',
guaranisignGaugeChartLefthalfRighthalf='rbxassetid://96206193501856',
guaranisignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://94122115224811',
guaranisignRing='rbxassetid://81261354529999',guaranisignRingDashed=
'rbxassetid://134424671572303',guaranisignSquare='rbxassetid://86259179384477',
guaranisignSquareFill='rbxassetid://110801531280759',guidepointHorizontal=
'rbxassetid://84781782029904',guidepointVertical='rbxassetid://75299633437864',
guidepointVerticalArrowtriangleForward='rbxassetid://136880401311404',
guidepointVerticalNumbers='rbxassetid://90754353774856',guitars=
'rbxassetid://89759348254657',guitarsFill='rbxassetid://118628784478150',
gyroscope='rbxassetid://87245230868142',hCircle='rbxassetid://79276843950798',
hCircleFill='rbxassetid://88400318014142',hSquare='rbxassetid://78518090497793',
hSquareFill='rbxassetid://119310296219192',hSquareOnSquare=
'rbxassetid://73872611085563',hSquareOnSquareFill='rbxassetid://87667186323197',
hammer='rbxassetid://112993982210428',hammerCircle=
'rbxassetid://112608453897290',hammerCircleFill='rbxassetid://115343834000229',
hammerFill='rbxassetid://102181818130125',handDraw=
'rbxassetid://117279982831998',handDrawBadgeEllipsis=
'rbxassetid://97969102000092',handDrawBadgeEllipsisFill=
'rbxassetid://139146605060373',handDrawFill='rbxassetid://138962845497785',
handPalmFacing='rbxassetid://78824669302390',handPalmFacingFill=
'rbxassetid://128639333246378',handPinch='rbxassetid://117668308815958',
handPinchFill='rbxassetid://133801146028572',handPointDown=
'rbxassetid://112615598975626',handPointDownFill='rbxassetid://105179986613891',
handPointLeft='rbxassetid://77661745009660',handPointLeftFill=
'rbxassetid://70525211956387',handPointRight='rbxassetid://97538565682591',
handPointRightFill='rbxassetid://121327486358231',handPointUp=
'rbxassetid://117889120409602',handPointUpBraille='rbxassetid://132543668022892'
,handPointUpBrailleBadgeEllipsis='rbxassetid://70655290798237',
handPointUpBrailleBadgeEllipsisFill='rbxassetid://88987354661672',
handPointUpBrailleFill='rbxassetid://138549174364488',handPointUpFill=
'rbxassetid://128174530086050',handPointUpLeft='rbxassetid://75447182366329',
handPointUpLeftAndText='rbxassetid://138503170903583',handPointUpLeftAndTextFill
='rbxassetid://97238879602996',handPointUpLeftFill='rbxassetid://89676540164824'
,handRaised='rbxassetid://84405411427687',handRaisedApp=
'rbxassetid://114135263702771',handRaisedAppFill='rbxassetid://90067463663710',
handRaisedBrakesignal='rbxassetid://137730513400429',handRaisedBrakesignalSlash=
'rbxassetid://100352691199559',handRaisedCircle='rbxassetid://116446437426242',
handRaisedCircleFill='rbxassetid://128584434185523',handRaisedFill=
'rbxassetid://78380006817166',handRaisedFingersSpread=
'rbxassetid://109788802602461',handRaisedFingersSpreadFill=
'rbxassetid://100765574389067',handRaisedPalmFacing=
'rbxassetid://97733489796350',handRaisedPalmFacingFill=
'rbxassetid://135020525701876',handRaisedSlash='rbxassetid://102308446798846',
handRaisedSlashFill='rbxassetid://103883906624498',handRaisedSquare=
'rbxassetid://139713537529028',handRaisedSquareFill=
'rbxassetid://132631307297758',handRaisedSquareOnSquare=
'rbxassetid://92160946199919',handRaisedSquareOnSquareFill=
'rbxassetid://107064045091001',handRays='rbxassetid://113827578510200',
handRaysFill='rbxassetid://81107317241495',handTap=
'rbxassetid://121078591974451',handTapFill='rbxassetid://107040362271120',
handThumbsdown='rbxassetid://86244158573902',handThumbsdownCircle=
'rbxassetid://131507665718313',handThumbsdownCircleFill=
'rbxassetid://72570466776202',handThumbsdownFill='rbxassetid://119254958080448',
handThumbsdownFilledHandThumbsup='rbxassetid://129312899514257',
handThumbsdownHandThumbsup='rbxassetid://137009505636508',
handThumbsdownHandThumbsupFill='rbxassetid://105367500517028',
handThumbsdownHandThumbsupFilled='rbxassetid://107178383384090',
handThumbsdownSlash='rbxassetid://79807256603775',handThumbsdownSlashFill=
'rbxassetid://74459649358982',handThumbsup='rbxassetid://128825696312922',
handThumbsupCircle='rbxassetid://103851740662333',handThumbsupCircleFill=
'rbxassetid://76110205867291',handThumbsupFill='rbxassetid://99068764072945',
handThumbsupSlash='rbxassetid://127685946159633',handThumbsupSlashFill=
'rbxassetid://125613315163126',handWave='rbxassetid://99003092549916',
handWaveFill='rbxassetid://121274581284216',handbag=
'rbxassetid://119008806614003',handbagCircle='rbxassetid://121699579950837',
handbagCircleFill='rbxassetid://119779307441201',handbagFill=
'rbxassetid://132621634669616',handbagSensorTagRadiowavesLeftAndRight=
'rbxassetid://120465117885302',handbagSensorTagRadiowavesLeftAndRightFill=
'rbxassetid://137202755346414',handsAndSparkles='rbxassetid://117743916588668',
handsAndSparklesFill='rbxassetid://79507225656287',handsClap=
'rbxassetid://132285028680230',handsClapFill='rbxassetid://109479694040359',
hanger='rbxassetid://110883707987214',hare='rbxassetid://117125651358452',
hareCircle='rbxassetid://81410012263740',hareCircleFill=
'rbxassetid://125075745793371',hareFill='rbxassetid://98813388968086',hatCap=
'rbxassetid://128042779395199',hatCapFill='rbxassetid://102509811181096',
hatWidebrim='rbxassetid://124186898685706',hatWidebrimFill=
'rbxassetid://86146755083367',hazardsign='rbxassetid://77958913474310',
hazardsignFill='rbxassetid://76749713980748',headProfileArrowForwardAndVisionPro
='rbxassetid://70678052534847',headlightDaytime='rbxassetid://84175387087613',
headlightDaytimeFill='rbxassetid://111429276707908',headlightFog=
'rbxassetid://109963635819226',headlightFogFill='rbxassetid://112552940723745',
headlightHighBeam='rbxassetid://74121554429267',headlightHighBeamFill=
'rbxassetid://137245916188907',headlightLowBeam='rbxassetid://125392142355830',
headlightLowBeamFill='rbxassetid://81479147676998',headphones=
'rbxassetid://132794493030809',headphonesCircle='rbxassetid://72972156489516',
headphonesCircleFill='rbxassetid://105101131622323',headphonesDots=
'rbxassetid://99563475077840',headphonesOverEar='rbxassetid://94686833776442',
headphonesSensorTagRadiowavesLeftAndRight='rbxassetid://90239603333031',
headphonesSensorTagRadiowavesLeftAndRightFill='rbxassetid://127570957315647',
headphonesSlash='rbxassetid://111042856026852',headset=
'rbxassetid://130413167339658',headsetCircle='rbxassetid://102874182756345',
headsetCircleFill='rbxassetid://99680766544161',hearingdeviceAndSignalMeter=
'rbxassetid://122423830802161',hearingdeviceAndSignalMeterFill=
'rbxassetid://101534259662449',hearingdeviceEar='rbxassetid://116592072523482',
hearingdeviceEarFill='rbxassetid://79485233563481',heart=
'rbxassetid://107768052948480',heartBadgeBolt='rbxassetid://108146593346669',
heartBadgeBoltFill='rbxassetid://89201775898085',heartBadgeBoltSlash=
'rbxassetid://101803582419656',heartBadgeBoltSlashFill=
'rbxassetid://115081294249357',heartCircle='rbxassetid://74612932649320',
heartCircleFill='rbxassetid://112366120879148',heartFill=
'rbxassetid://133334197582974',heartGaugeOpen='rbxassetid://83392107700158',
heartRectangle='rbxassetid://97775143423887',heartRectangleFill=
'rbxassetid://81040147605111',heartSlash='rbxassetid://102340079577590',
heartSlashCircle='rbxassetid://105081950550053',heartSlashCircleFill=
'rbxassetid://117503908060806',heartSlashFill='rbxassetid://98288490017510',
heartSquare='rbxassetid://112292843005652',heartSquareFill=
'rbxassetid://113665033930324',heartTextClipboard='rbxassetid://135119715100910'
,heartTextClipboardFill='rbxassetid://97188049534337',heartTextSquare=
'rbxassetid://74020211110629',heartTextSquareFill='rbxassetid://96589545065469',
heatElementWindshield='rbxassetid://85064985076162',heatWaves=
'rbxassetid://117183214242618',heatWavesAndFan='rbxassetid://131275641266480',
heatWavesCircle='rbxassetid://107780936807304',heatWavesCircleFill=
'rbxassetid://124151920948426',heatWavesGaugeOpen='rbxassetid://109476160740075'
,heaterVertical='rbxassetid://72918931799735',heaterVerticalFill=
'rbxassetid://92635624311644',helm='rbxassetid://128890437419395',helmet=
'rbxassetid://114891866790157',helmetFill='rbxassetid://119174880478810',hexagon
='rbxassetid://121821922240266',hexagonBottomhalfFilled=
'rbxassetid://111107059226028',hexagonFill='rbxassetid://82240827497051',
hexagonLefthalfFilled='rbxassetid://101561513084357',hexagonRighthalfFilled=
'rbxassetid://107562615063917',hexagonTophalfFilled=
'rbxassetid://132335379358202',hifireceiver='rbxassetid://135361029083378',
hifireceiverFill='rbxassetid://99759386860407',hifispeaker=
'rbxassetid://99164524327344',hifispeaker2='rbxassetid://86111767037300',
hifispeaker2BadgeMinus='rbxassetid://118971920951758',hifispeaker2BadgeMinusFill
='rbxassetid://101665851094245',hifispeaker2BadgePlus=
'rbxassetid://115961834970221',hifispeaker2BadgePlusFill=
'rbxassetid://72006888131536',hifispeaker2Fill='rbxassetid://88815689838039',
hifispeakerAndAppletv='rbxassetid://127147046090615',hifispeakerAndAppletvFill=
'rbxassetid://83573337866673',hifispeakerAndHomepod=
'rbxassetid://82595174597082',hifispeakerAndHomepodBadgeMinus=
'rbxassetid://112609157830200',hifispeakerAndHomepodBadgeMinusFill=
'rbxassetid://74023887981775',hifispeakerAndHomepodBadgePlus=
'rbxassetid://85291697963379',hifispeakerAndHomepodBadgePlusFill=
'rbxassetid://123316634293817',hifispeakerAndHomepodFill=
'rbxassetid://73377066571876',hifispeakerAndHomepodMini=
'rbxassetid://70934710071244',hifispeakerAndHomepodMiniBadgeMinus=
'rbxassetid://78650669958266',hifispeakerAndHomepodMiniBadgeMinusFill=
'rbxassetid://130888989200488',hifispeakerAndHomepodMiniBadgePlus=
'rbxassetid://101156627050743',hifispeakerAndHomepodMiniBadgePlusFill=
'rbxassetid://72856342595422',hifispeakerAndHomepodMiniFill=
'rbxassetid://124757037235263',hifispeakerArrowForward=
'rbxassetid://106004965331315',hifispeakerArrowForwardFill=
'rbxassetid://109292392722010',hifispeakerBadgeMinus=
'rbxassetid://113543299876891',hifispeakerBadgeMinusFill=
'rbxassetid://106840703331361',hifispeakerBadgePlus=
'rbxassetid://86077612672916',hifispeakerBadgePlusFill=
'rbxassetid://79039318209871',hifispeakerFill='rbxassetid://77801175030655',
highlighter='rbxassetid://77046693747579',highlighterBadgeEllipsis=
'rbxassetid://90559499376399',hockeyPuck='rbxassetid://82861679405783',
hockeyPuckCircle='rbxassetid://74508151585209',hockeyPuckCircleFill=
'rbxassetid://83552419639351',hockeyPuckFill='rbxassetid://112221361054966',
holdBrakesignal='rbxassetid://93719484889123',homepod=
'rbxassetid://138665920971606',homepod2='rbxassetid://124703967838105',
homepod2BadgeMinus='rbxassetid://72010862188560',homepod2BadgeMinusFill=
'rbxassetid://122422111268643',homepod2BadgePlus='rbxassetid://95527221485415',
homepod2BadgePlusFill='rbxassetid://124657181184492',homepod2Fill=
'rbxassetid://71022507008871',homepodAndAppletv='rbxassetid://137717079586958',
homepodAndAppletvFill='rbxassetid://109445378694806',homepodAndHomepodMini=
'rbxassetid://121344187904185',homepodAndHomepodMiniBadgeMinus=
'rbxassetid://111143025398497',homepodAndHomepodMiniBadgeMinusFill=
'rbxassetid://136855742899911',homepodAndHomepodMiniBadgePlus=
'rbxassetid://129503453741868',homepodAndHomepodMiniBadgePlusFill=
'rbxassetid://73815090550857',homepodAndHomepodMiniFill=
'rbxassetid://82957016359447',homepodArrowForward='rbxassetid://112827593003603'
,homepodArrowForwardFill='rbxassetid://111457680101526',homepodBadgeCheckmark=
'rbxassetid://82279608071963',homepodBadgeCheckmarkFill=
'rbxassetid://119126537015840',homepodBadgeMinus='rbxassetid://111844922298587',
homepodBadgeMinusFill='rbxassetid://114830828502660',homepodBadgePlus=
'rbxassetid://106645926660190',homepodBadgePlusFill=
'rbxassetid://89951217734368',homepodFill='rbxassetid://126573700088988',
homepodMini='rbxassetid://119380942973091',homepodMini2=
'rbxassetid://119573077798052',homepodMini2BadgeMinus=
'rbxassetid://75537675870052',homepodMini2BadgeMinusFill=
'rbxassetid://89666757230874',homepodMini2BadgePlus=
'rbxassetid://131253222073711',homepodMini2BadgePlusFill=
'rbxassetid://122139521655674',homepodMini2Fill='rbxassetid://129534319223398',
homepodMiniAndAppletv='rbxassetid://126403464830823',homepodMiniAndAppletvFill=
'rbxassetid://102930412180504',homepodMiniArrowForward=
'rbxassetid://103858774531499',homepodMiniArrowForwardFill=
'rbxassetid://88548291845376',homepodMiniBadgeCheckmark=
'rbxassetid://95602217269613',homepodMiniBadgeCheckmarkFill=
'rbxassetid://138452597685198',homepodMiniBadgeMinus=
'rbxassetid://119379139369139',homepodMiniBadgeMinusFill=
'rbxassetid://126293519479293',homepodMiniBadgePlus=
'rbxassetid://118379467986303',homepodMiniBadgePlusFill=
'rbxassetid://111517543574290',homepodMiniFill='rbxassetid://115748763010662',
horn='rbxassetid://133092229616767',hornBlast='rbxassetid://114595846337741',
hornBlastFill='rbxassetid://101724439604991',hornFill=
'rbxassetid://135647807274226',hourglass='rbxassetid://85321944407184',
hourglassBadgeEye='rbxassetid://135319595841957',hourglassBadgeLock=
'rbxassetid://76338330289232',hourglassBadgePlus='rbxassetid://114268852033821',
hourglassBottomhalfFilled='rbxassetid://113354322030926',hourglassCircle=
'rbxassetid://110733468934387',hourglassCircleFill=
'rbxassetid://125048992318731',hourglassTophalfFilled=
'rbxassetid://84671059736173',house='rbxassetid://137977066267668',houseAndFlag=
'rbxassetid://86134823093935',houseAndFlagCircle='rbxassetid://127330265020154',
houseAndFlagCircleFill='rbxassetid://70711300093336',houseAndFlagFill=
'rbxassetid://109752212574467',houseBadgeExclamationmark=
'rbxassetid://76429262032538',houseBadgeExclamationmarkFill=
'rbxassetid://87392853954669',houseBadgeWifi='rbxassetid://135608851537692',
houseBadgeWifiFill='rbxassetid://103445459783943',houseCircle=
'rbxassetid://138648903313463',houseCircleFill='rbxassetid://108559873908847',
houseFill='rbxassetid://139946019827392',houseLodge=
'rbxassetid://85704833980676',houseLodgeCircle='rbxassetid://127650865114252',
houseLodgeCircleFill='rbxassetid://71323700043692',houseLodgeFill=
'rbxassetid://127089968967039',houseSlash='rbxassetid://81400788586332',
houseSlashFill='rbxassetid://137443750353131',hryvniasign=
'rbxassetid://103668150355749',
hryvniasignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://91688749623067',hryvniasignBankBuilding=
'rbxassetid://129621086591675',hryvniasignBankBuildingFill=
'rbxassetid://104628622071039',hryvniasignCircle='rbxassetid://94474243422007',
hryvniasignCircleFill='rbxassetid://101794866939417',
hryvniasignGaugeChartLefthalfRighthalf='rbxassetid://106063402550840',
hryvniasignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://128998826362994',
hryvniasignRing='rbxassetid://118368396656485',hryvniasignRingDashed=
'rbxassetid://88816425414859',hryvniasignSquare='rbxassetid://75385636486518',
hryvniasignSquareFill='rbxassetid://89586571568331',humidifier=
'rbxassetid://77734577476092',humidifierAndDroplets=
'rbxassetid://109490683520378',humidifierAndDropletsFill=
'rbxassetid://81835383574136',humidifierAndEllipsis=
'rbxassetid://119958476085943',humidifierAndEllipsisFill=
'rbxassetid://107200405950324',humidifierFill='rbxassetid://127532491737560',
humidity='rbxassetid://94944682462623',humidityFill=
'rbxassetid://116083322385268',hurricane='rbxassetid://95533741958668',
hurricaneCircle='rbxassetid://90440100760735',hurricaneCircleFill=
'rbxassetid://130652138255798',hydrogen='rbxassetid://92886127488787',
hydrogenCircle='rbxassetid://86547013940983',hydrogenCircleFill=
'rbxassetid://89516381077394',hydrogenSquare='rbxassetid://139351374466565',
hydrogenSquareFill='rbxassetid://137173913274988',iCircle=
'rbxassetid://89113344291580',iCircleFill='rbxassetid://104332397605196',iSquare
='rbxassetid://96564441985200',iSquareFill='rbxassetid://113037407421563',icloud
='rbxassetid://110727461515621',icloudAndArrowDown=
'rbxassetid://133728071871190',icloudAndArrowDownFill=
'rbxassetid://91580842064758',icloudAndArrowUp='rbxassetid://82876886918286',
icloudAndArrowUpFill='rbxassetid://80667605488140',icloudCircle=
'rbxassetid://130045045841352',icloudCircleFill='rbxassetid://139289409047379',
icloudDashed='rbxassetid://111793754917716',icloudFill=
'rbxassetid://140102319448039',icloudSlash='rbxassetid://77090637996631',
icloudSlashFill='rbxassetid://117065145373314',icloudSquare=
'rbxassetid://102626362675694',icloudSquareFill='rbxassetid://97104665653244',
increaseIndent='rbxassetid://88800434030947',increaseQuotelevel=
'rbxassetid://104026428447630',indianrupeesign='rbxassetid://111754186050334',
indianrupeesignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://112994062871958',indianrupeesignBankBuilding=
'rbxassetid://119082346279444',indianrupeesignBankBuildingFill=
'rbxassetid://92479240380407',indianrupeesignCircle=
'rbxassetid://123695503681843',indianrupeesignCircleFill=
'rbxassetid://129318061897341',indianrupeesignGaugeChartLefthalfRighthalf=
'rbxassetid://78225158661344',
indianrupeesignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://82863857678210',indianrupeesignRing='rbxassetid://75461897769861',
indianrupeesignRingDashed='rbxassetid://84664976795938',indianrupeesignSquare=
'rbxassetid://122397780894486',indianrupeesignSquareFill=
'rbxassetid://105131855370809',infinity='rbxassetid://71263460362433',
infinityCircle='rbxassetid://71003157985047',infinityCircleFill=
'rbxassetid://86156332621236',info='rbxassetid://119116685807529',infoBubble=
'rbxassetid://137273126608579',infoBubbleFill='rbxassetid://124693206931792',
infoCircle='rbxassetid://140318155547831',infoCircleFill=
'rbxassetid://120424810493200',infoCircleTextPage='rbxassetid://125356901438011'
,infoCircleTextPageFill='rbxassetid://80370826247197',infoSquare=
'rbxassetid://110012911332380',infoSquareFill='rbxassetid://95776862176830',
infoTriangle='rbxassetid://94781493039380',infoTriangleFill=
'rbxassetid://95650689777625',infoWindshield='rbxassetid://128644339648564',
inhaler='rbxassetid://121743497043663',inhalerFill='rbxassetid://71451949590628'
,insetFilledApplewatchCase='rbxassetid://120279724059828',
insetFilledBottomhalfRectangle='rbxassetid://136597041452039',
insetFilledBottomhalfRectanglePortrait='rbxassetid://100868664232614',
insetFilledBottomhalfTophalfRectangle='rbxassetid://127692599846777',
insetFilledBottomleadingBottomtrailingRectangle='rbxassetid://84327980842976',
insetFilledBottomleadingRectangle='rbxassetid://101305622798572',
insetFilledBottomleadingRectanglePortrait='rbxassetid://139106808860498',
insetFilledBottomleftBottomrightRectangle='rbxassetid://135473749561029',
insetFilledBottomleftRectangle='rbxassetid://96902285478363',
insetFilledBottomleftRectanglePortrait='rbxassetid://128299495559712',
insetFilledBottomrightRectangle='rbxassetid://120529833708590',
insetFilledBottomrightRectanglePortrait='rbxassetid://126122940689687',
insetFilledBottomthirdRectangle='rbxassetid://92508256484902',
insetFilledBottomthirdRectanglePortrait='rbxassetid://132794319919125',
insetFilledBottomthirdSquare='rbxassetid://76623279068770',
insetFilledBottomtrailingRectangle='rbxassetid://121412469782708',
insetFilledBottomtrailingRectanglePortrait='rbxassetid://93668101678869',
insetFilledCapsule='rbxassetid://90445157526338',insetFilledCapsulePortrait=
'rbxassetid://108299551226678',insetFilledCenterRectangle=
'rbxassetid://98977891812691',insetFilledCenterRectangleBadgePlus=
'rbxassetid://104683887573848',insetFilledCenterRectanglePortrait=
'rbxassetid://73922453045894',insetFilledCircle='rbxassetid://75917977120341',
insetFilledCircleDashed='rbxassetid://116630264943051',insetFilledCircleSlash=
'rbxassetid://121268175510312',insetFilledDiamond='rbxassetid://126741449338194'
,insetFilledLeadinghalfArrowLeadingRectangle='rbxassetid://78017625118769',
insetFilledLeadinghalfRectangle='rbxassetid://130706184301016',
insetFilledLeadinghalfRectanglePortrait='rbxassetid://85919286811701',
insetFilledLeadinghalfToptrailingBottomtrailingRectangle=
'rbxassetid://70494881048997',insetFilledLeadinghalfTrailinghalfRectangle=
'rbxassetid://134274869575399',insetFilledLeadingthirdRectangle=
'rbxassetid://136915676163117',insetFilledLeadingthirdRectanglePortrait=
'rbxassetid://113702527307183',insetFilledLeadingthirdSquare=
'rbxassetid://109950316628586',insetFilledLefthalfArrowLeftRectangle=
'rbxassetid://134984195372501',insetFilledLefthalfRectangle=
'rbxassetid://72023436186359',insetFilledLefthalfRectanglePortrait=
'rbxassetid://120313318229969',insetFilledLefthalfRighthalfRectangle=
'rbxassetid://102132835584174',insetFilledLefthalfToprightBottomrightRectangle=
'rbxassetid://111841202861656',
insetFilledLeftthirdMiddlethirdRightthirdRectangle='rbxassetid://77699442899450'
,insetFilledLeftthirdRectangle='rbxassetid://112754738404469',
insetFilledLeftthirdRectanglePortrait='rbxassetid://131316644740016',
insetFilledLeftthirdSquare='rbxassetid://122568783815641',insetFilledOval=
'rbxassetid://111278423834459',insetFilledOvalPortrait=
'rbxassetid://125501917035259',insetFilledPano='rbxassetid://80590947649666',
insetFilledRectangle='rbxassetid://138708382163165',
insetFilledRectangleAndPersonFilled='rbxassetid://87154458723474',
insetFilledRectangleAndPersonFilledCircle='rbxassetid://95193435746744',
insetFilledRectangleAndPersonFilledCircleFill='rbxassetid://71284408528556',
insetFilledRectangleAndPointerArrow='rbxassetid://71367104401668',
insetFilledRectangleBadgeRecord='rbxassetid://105070640753020',
insetFilledRectangleOnRectangle='rbxassetid://120205174258551',
insetFilledRectanglePortrait='rbxassetid://100405460391582',
insetFilledRighthalfArrowRightRectangle='rbxassetid://136171522595126',
insetFilledRighthalfLefthalfRectangle='rbxassetid://131735122239469',
insetFilledRighthalfRectangle='rbxassetid://129492638501890',
insetFilledRighthalfRectanglePortrait='rbxassetid://125346899504670',
insetFilledRightthirdRectangle='rbxassetid://70604242598186',
insetFilledRightthirdRectanglePortrait='rbxassetid://105069897135082',
insetFilledRightthirdSquare='rbxassetid://104572444958233',insetFilledSquare=
'rbxassetid://133012340169358',insetFilledSquareDashed=
'rbxassetid://83067474683427',insetFilledTophalfBottomhalfRectangle=
'rbxassetid://105407482136406',insetFilledTophalfBottomleftBottomrightRectangle=
'rbxassetid://126213084936540',insetFilledTophalfRectangle=
'rbxassetid://102954017318491',insetFilledTophalfRectanglePortrait=
'rbxassetid://96770534088202',
insetFilledTopleadingBottomleadingTrailinghalfRectangle=
'rbxassetid://133635573770248',insetFilledTopleadingRectangle=
'rbxassetid://82767537329467',insetFilledTopleadingRectanglePortrait=
'rbxassetid://103128241995328',insetFilledTopleftBottomleftRighthalfRectangle=
'rbxassetid://136216039981584',insetFilledTopleftRectangle=
'rbxassetid://126843838930696',insetFilledTopleftRectanglePortrait=
'rbxassetid://96032033362183',insetFilledTopleftToprightBottomhalfRectangle=
'rbxassetid://131903028482382',
insetFilledTopleftToprightBottomleftBottomrightRectangle=
'rbxassetid://108794580005907',insetFilledToprightRectangle=
'rbxassetid://128249878170636',insetFilledToprightRectanglePortrait=
'rbxassetid://81500237959732',insetFilledTopthirdMiddlethirdBottomthirdRectangle
='rbxassetid://125282916436565',insetFilledTopthirdRectangle=
'rbxassetid://113341401260201',insetFilledTopthirdRectanglePortrait=
'rbxassetid://120512386877930',insetFilledTopthirdSquare=
'rbxassetid://119824385876092',insetFilledToptrailingRectangle=
'rbxassetid://129029043300013',insetFilledToptrailingRectanglePortrait=
'rbxassetid://100995703293630',insetFilledTrailinghalfArrowTrailingRectangle=
'rbxassetid://84836774081264',insetFilledTrailinghalfLeadinghalfRectangle=
'rbxassetid://103401929210623',insetFilledTrailinghalfRectangle=
'rbxassetid://133451001758776',insetFilledTrailinghalfRectanglePortrait=
'rbxassetid://112562041419219',insetFilledTrailingthirdRectangle=
'rbxassetid://118636064884027',insetFilledTrailingthirdRectanglePortrait=
'rbxassetid://75897427033432',insetFilledTrailingthirdSquare=
'rbxassetid://114005042149867',insetFilledTriangle=
'rbxassetid://121531632404240',insetFilledTv='rbxassetid://109545088046904',
internaldrive='rbxassetid://102251712322050',internaldriveFill=
'rbxassetid://75514367271798',ipad='rbxassetid://131230554306229',
ipadAndArrowForward='rbxassetid://110049781892554',ipadBadgeCheckmark=
'rbxassetid://123793516179129',ipadBadgeExclamationmark=
'rbxassetid://113541137806339',ipadBadgeLocation='rbxassetid://117276337518182',
ipadBadgePlay='rbxassetid://135586545630005',ipadCase=
'rbxassetid://72648998001913',ipadCaseAndIphoneCase=
'rbxassetid://105917542288684',ipadGen1='rbxassetid://123061750908863',
ipadGen1BadgeExclamationmark='rbxassetid://97502161805065',ipadGen1BadgeLocation
='rbxassetid://90300741766128',ipadGen1BadgePlay='rbxassetid://135252945335606',
ipadGen1CropHomebuttonCircle='rbxassetid://123435089120942',ipadGen1Landscape=
'rbxassetid://81891741408956',ipadGen1LandscapeBadgeExclamationmark=
'rbxassetid://107044013715078',ipadGen1LandscapeBadgeLocation=
'rbxassetid://124387888974099',ipadGen1LandscapeBadgePlay=
'rbxassetid://81439451813082',ipadGen1LandscapeSlash=
'rbxassetid://115395749352607',ipadGen1Sizes='rbxassetid://118294310937760',
ipadGen1Slash='rbxassetid://137094723034266',ipadGen2=
'rbxassetid://72845801035243',ipadGen2BadgeExclamationmark=
'rbxassetid://86689553406739',ipadGen2BadgeLocation=
'rbxassetid://123318608980440',ipadGen2BadgePlay='rbxassetid://120431580583179',
ipadGen2Landscape='rbxassetid://73336322338582',
ipadGen2LandscapeBadgeExclamationmark='rbxassetid://74543577579883',
ipadGen2LandscapeBadgeLocation='rbxassetid://72978217701125',
ipadGen2LandscapeBadgePlay='rbxassetid://88880081274171',ipadGen2LandscapeSlash=
'rbxassetid://93298411493155',ipadGen2Sizes='rbxassetid://92860208821978',
ipadGen2Slash='rbxassetid://97413632843279',ipadLandscape=
'rbxassetid://98570913676933',ipadLandscapeAndApplewatch=
'rbxassetid://78572604978981',ipadLandscapeAndIphone=
'rbxassetid://103405408625627',ipadLandscapeAndIphoneSlash=
'rbxassetid://83949384569338',ipadLandscapeAndIpod=
'rbxassetid://126361538286391',ipadLandscapeBadgeExclamationmark=
'rbxassetid://79735738496284',ipadLandscapeBadgeLocation=
'rbxassetid://119487770282833',ipadLandscapeBadgePlay=
'rbxassetid://131593550438106',ipadRearCamera='rbxassetid://100950321161338',
ipadSizes='rbxassetid://117365106864126',iphone='rbxassetid://96785236033941',
iphoneAndArrowForwardInward='rbxassetid://127905379728905',
iphoneAndArrowForwardOutward='rbxassetid://80118778349170',
iphoneAndArrowLeftAndArrowRightInward='rbxassetid://117948708918605',
iphoneAndArrowRightInward='rbxassetid://79888523696187',
iphoneAndArrowRightOutward='rbxassetid://122240049510090',iphoneAndIpod=
'rbxassetid://97511296416293',iphoneAndVisionPro='rbxassetid://120484375912051',
iphoneAppSwitcher='rbxassetid://133135376021847',iphoneBadgeCheckmark=
'rbxassetid://102860706153917',iphoneBadgeExclamationmark=
'rbxassetid://71713794425391',iphoneBadgeLocation='rbxassetid://122605433839128'
,iphoneBadgePlay='rbxassetid://99553977604210',iphoneCase=
'rbxassetid://100408903353145',iphoneCircle='rbxassetid://136014152812869',
iphoneCircleFill='rbxassetid://117758615235459',iphoneCropCircle=
'rbxassetid://71562894009267',iphoneDockMotorizedViewfinder=
'rbxassetid://132637320368921',iphoneGen1='rbxassetid://94152158848001',
iphoneGen1AndArrowLeft='rbxassetid://127049565346505',
iphoneGen1BadgeExclamationmark='rbxassetid://77237650629580',
iphoneGen1BadgeLocation='rbxassetid://104241024785309',iphoneGen1BadgePlay=
'rbxassetid://92461645076483',iphoneGen1Circle='rbxassetid://116798852794419',
iphoneGen1CircleFill='rbxassetid://94921719612712',iphoneGen1CropCircle=
'rbxassetid://105220191366334',iphoneGen1CropHomebuttonCircle=
'rbxassetid://120481565169469',iphoneGen1Landscape=
'rbxassetid://137867981805693',iphoneGen1LandscapeSlash=
'rbxassetid://128717452111489',iphoneGen1Motion='rbxassetid://122054190796509',
iphoneGen1RadiowavesLeftAndRight='rbxassetid://133529445742502',
iphoneGen1RadiowavesLeftAndRightCircle='rbxassetid://125660794021542',
iphoneGen1RadiowavesLeftAndRightCircleFill='rbxassetid://106520977934691',
iphoneGen1Sizes='rbxassetid://133043833497584',iphoneGen1Slash=
'rbxassetid://132512376385662',iphoneGen1SlashCircle=
'rbxassetid://131363565228610',iphoneGen1SlashCircleFill=
'rbxassetid://139459042705882',iphoneGen2='rbxassetid://126158544332994',
iphoneGen2AndArrowLeftAndArrowRightInward='rbxassetid://116971900100346',
iphoneGen2BadgeExclamationmark='rbxassetid://124222723580362',
iphoneGen2BadgeLocation='rbxassetid://89458034440222',iphoneGen2BadgePlay=
'rbxassetid://83586848263184',iphoneGen2Circle='rbxassetid://84600552457895',
iphoneGen2CircleFill='rbxassetid://102360021492589',iphoneGen2CropCircle=
'rbxassetid://86170718962353',iphoneGen2Landscape='rbxassetid://90012948331103',
iphoneGen2LandscapeSlash='rbxassetid://107053953408192',iphoneGen2Motion=
'rbxassetid://75209858125378',iphoneGen2RadiowavesLeftAndRight=
'rbxassetid://137843293377967',iphoneGen2RadiowavesLeftAndRightCircle=
'rbxassetid://78517623289183',iphoneGen2RadiowavesLeftAndRightCircleFill=
'rbxassetid://131656745166889',iphoneGen2Sizes='rbxassetid://81272476589028',
iphoneGen2Slash='rbxassetid://76336416899694',iphoneGen2SlashCircle=
'rbxassetid://133859389260522',iphoneGen2SlashCircleFill=
'rbxassetid://87199768439188',iphoneGen3='rbxassetid://92492572556324',
iphoneGen3AndArrowLeftAndArrowRightInward='rbxassetid://118657589793721',
iphoneGen3BadgeExclamationmark='rbxassetid://113121813406073',
iphoneGen3BadgeLocation='rbxassetid://117982542340771',iphoneGen3BadgePlay=
'rbxassetid://125330422338100',iphoneGen3Circle='rbxassetid://86912097932476',
iphoneGen3CircleFill='rbxassetid://94525449390144',iphoneGen3CropCircle=
'rbxassetid://94048325413237',iphoneGen3Landscape='rbxassetid://114233975397990'
,iphoneGen3LandscapeSlash='rbxassetid://132992124593122',iphoneGen3Motion=
'rbxassetid://98169599088581',iphoneGen3RadiowavesLeftAndRight=
'rbxassetid://72200807533817',iphoneGen3RadiowavesLeftAndRightCircle=
'rbxassetid://106374321761914',iphoneGen3RadiowavesLeftAndRightCircleFill=
'rbxassetid://72465338711976',iphoneGen3Sizes='rbxassetid://76366535677582',
iphoneGen3Slash='rbxassetid://90077097271339',iphoneGen3SlashCircle=
'rbxassetid://111474089697962',iphoneGen3SlashCircleFill=
'rbxassetid://111157278153150',iphoneLandscape='rbxassetid://127446547566888',
iphoneMotion='rbxassetid://130536425029393',iphonePatternDiagonalline=
'rbxassetid://135162003513277',
iphonePatternDiagonallineOnRectanglePortraitDashed=
'rbxassetid://112745590711907',iphoneRadiowavesLeftAndRight=
'rbxassetid://100186481961591',iphoneRadiowavesLeftAndRightCircle=
'rbxassetid://124241232816642',iphoneRadiowavesLeftAndRightCircleFill=
'rbxassetid://120377260746135',iphoneRearCamera='rbxassetid://120522581577378',
iphoneSizes='rbxassetid://135407341947073',iphoneSlash=
'rbxassetid://90628537365176',iphoneSlashCircle='rbxassetid://122677491004031',
iphoneSlashCircleFill='rbxassetid://115785875771965',iphoneSmartbatterycaseGen1=
'rbxassetid://134849352121716',iphoneSmartbatterycaseGen2=
'rbxassetid://85196761349659',ipod='rbxassetid://80468906468753',
ipodAndApplewatch='rbxassetid://93719301646294',ipodAndVisionPro=
'rbxassetid://83017359796219',ipodShuffleGen1='rbxassetid://136172132310552',
ipodShuffleGen2='rbxassetid://76170036384439',ipodShuffleGen3=
'rbxassetid://110263498432868',ipodShuffleGen4='rbxassetid://131032367315049',
ipodTouch='rbxassetid://105267613849911',ipodTouchLandscape=
'rbxassetid://116094116999635',ipodTouchSlash='rbxassetid://81927156094607',
italic='rbxassetid://114841606569634',ivfluidBag='rbxassetid://85925137715594',
ivfluidBagFill='rbxassetid://112534881575448',jCircle=
'rbxassetid://113787719639007',jCircleFill='rbxassetid://136066715693266',
jSquare='rbxassetid://132663369942785',jSquareFill=
'rbxassetid://110186942350601',jSquareOnSquare='rbxassetid://70770700564140',
jSquareOnSquareFill='rbxassetid://86732752263868',jacket=
'rbxassetid://110977300370127',jacketCircle='rbxassetid://73905147156647',
jacketCircleFill='rbxassetid://87569696632536',jacketFill=
'rbxassetid://129429549539056',jacketSensorTagRadiowavesLeftAndRight=
'rbxassetid://81170082341074',jacketSensorTagRadiowavesLeftAndRightFill=
'rbxassetid://86875701016809',k='rbxassetid://131453804995230',kCircle=
'rbxassetid://108260281955032',kCircleFill='rbxassetid://127917254878881',
kSquare='rbxassetid://76333957199097',kSquareFill='rbxassetid://109715448814787'
,kashidaArabic='rbxassetid://113587154651306',key='rbxassetid://133434192543807'
,key2OnRing='rbxassetid://76703793889414',key2OnRingFill=
'rbxassetid://76588755997300',keyCarRadiowavesForward=
'rbxassetid://112640775536914',keyCarRadiowavesForwardFill=
'rbxassetid://113133901858914',keyCarSide='rbxassetid://130777080754266',
keyCarSideFill='rbxassetid://97814070412964',keyCard=
'rbxassetid://115187278538124',keyCardFill='rbxassetid://139348357957386',
keyCircle='rbxassetid://108973244339496',keyCircleFill=
'rbxassetid://110326981381894',keyConvertibleSide='rbxassetid://89421482992474',
keyConvertibleSideFill='rbxassetid://72129430413908',keyFill=
'rbxassetid://100941145516940',keyHorizontal='rbxassetid://136772885372533',
keyHorizontalFill='rbxassetid://120752932235278',keyIcloud=
'rbxassetid://139241478885890',keyIcloudFill='rbxassetid://72564816797669',
keyRadiowavesForward='rbxassetid://120443557468351',keyRadiowavesForwardFill=
'rbxassetid://73268293809232',keyRadiowavesForwardSlash=
'rbxassetid://91223085154439',keyRadiowavesForwardSlashFill=
'rbxassetid://131793789575248',keySensorTagRadiowavesLeftAndRight=
'rbxassetid://96659898144332',keySensorTagRadiowavesLeftAndRightFill=
'rbxassetid://98758622563103',keyShield='rbxassetid://104889894182978',
keyShieldFill='rbxassetid://115236492425793',keySlash=
'rbxassetid://128541901933913',keySlashFill='rbxassetid://103334949123430',
keySuvSide='rbxassetid://113533289572731',keySuvSideFill=
'rbxassetid://98960872652240',keyTruckPickupSide='rbxassetid://140618494739761',
keyTruckPickupSideFill='rbxassetid://72634089700259',keyViewfinder=
'rbxassetid://102804665349975',keyboard='rbxassetid://105919136330989',
keyboardBadgeEllipsis='rbxassetid://138764951588511',keyboardBadgeEllipsisFill=
'rbxassetid://129826695434297',keyboardBadgeEye='rbxassetid://88947978884486',
keyboardBadgeEyeFill='rbxassetid://113136519679576',keyboardChevronCompactDown=
'rbxassetid://122434132254331',keyboardChevronCompactDownFill=
'rbxassetid://132119530733764',keyboardChevronCompactLeft=
'rbxassetid://93873100176471',keyboardChevronCompactLeftFill=
'rbxassetid://122102681496501',keyboardFill='rbxassetid://107094757259919',
keyboardMacwindow='rbxassetid://81009841388075',keyboardOnehandedLeft=
'rbxassetid://96435588655591',keyboardOnehandedLeftFill=
'rbxassetid://84926357433136',keyboardOnehandedRight=
'rbxassetid://99512020372077',keyboardOnehandedRightFill=
'rbxassetid://114811778842920',kipsign='rbxassetid://139124096600858',
kipsignArrowTriangleheadCounterclockwiseRotate90='rbxassetid://121587106422922',
kipsignBankBuilding='rbxassetid://118184717903574',kipsignBankBuildingFill=
'rbxassetid://124188099989927',kipsignCircle='rbxassetid://125257177249690',
kipsignCircleFill='rbxassetid://87946535077535',
kipsignGaugeChartLefthalfRighthalf='rbxassetid://132967067870371',
kipsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://73726622341666',
kipsignRing='rbxassetid://99797321226540',kipsignRingDashed=
'rbxassetid://105404390233593',kipsignSquare='rbxassetid://133836203040078',
kipsignSquareFill='rbxassetid://80235335357450',kph=
'rbxassetid://117481583715113',kphCircle='rbxassetid://137148268988469',
kphCircleFill='rbxassetid://126101024239774',l1ButtonRoundedbottomHorizontal=
'rbxassetid://95790368795483',l1ButtonRoundedbottomHorizontalFill=
'rbxassetid://128440014143027',l1Circle='rbxassetid://102236186726427',
l1CircleFill='rbxassetid://116773512326026',l2ButtonAngledtopVerticalLeft=
'rbxassetid://99495903296844',l2ButtonAngledtopVerticalLeftFill=
'rbxassetid://110271917565119',l2ButtonRoundedtopHorizontal=
'rbxassetid://91390095902654',l2ButtonRoundedtopHorizontalFill=
'rbxassetid://79761826766984',l2Circle='rbxassetid://74248236678335',
l2CircleFill='rbxassetid://109925799906152',l3ButtonAngledbottomHorizontalLeft=
'rbxassetid://123803597683885',l3ButtonAngledbottomHorizontalLeftFill=
'rbxassetid://134808114449129',l4ButtonHorizontal='rbxassetid://100597686733238'
,l4ButtonHorizontalFill='rbxassetid://89171783331888',
lButtonRoundedbottomHorizontal='rbxassetid://130058952204010',
lButtonRoundedbottomHorizontalFill='rbxassetid://105492940635087',lCircle=
'rbxassetid://88768237168264',lCircleFill='rbxassetid://115654171291345',
lJoystick='rbxassetid://124158836435588',lJoystickFill=
'rbxassetid://72524976304902',lJoystickPressDown='rbxassetid://126988825422704',
lJoystickPressDownFill='rbxassetid://122290771310900',lJoystickTiltDown=
'rbxassetid://98216370037941',lJoystickTiltDownFill=
'rbxassetid://106818736496317',lJoystickTiltLeft='rbxassetid://117338893439682',
lJoystickTiltLeftFill='rbxassetid://115979752014913',lJoystickTiltRight=
'rbxassetid://104661243170230',lJoystickTiltRightFill=
'rbxassetid://127932461706178',lJoystickTiltUp='rbxassetid://103418791124625',
lJoystickTiltUpFill='rbxassetid://139059621301014',lSquare=
'rbxassetid://135675870127012',lSquareFill='rbxassetid://92188040928529',ladybug
='rbxassetid://106759060151131',ladybugCircle='rbxassetid://79448683961001',
ladybugCircleFill='rbxassetid://133960361550152',ladybugFill=
'rbxassetid://113818385620830',ladybugSlash='rbxassetid://122118980779982',
ladybugSlashCircle='rbxassetid://79524457814539',ladybugSlashCircleFill=
'rbxassetid://135161883645329',ladybugSlashFill='rbxassetid://90687484782825',
lampCeiling='rbxassetid://130697710320625',lampCeilingFill=
'rbxassetid://88194539041083',lampCeilingInverse='rbxassetid://94283461619651',
lampDesk='rbxassetid://103868701476630',lampDeskFill=
'rbxassetid://131947905513117',lampFloor='rbxassetid://118272420344055',
lampFloorFill='rbxassetid://81002667912169',lampTable=
'rbxassetid://131486052019715',lampTableFill='rbxassetid://78919597241075',lane=
'rbxassetid://86601744938682',lanyardcard='rbxassetid://125357233986697',
lanyardcardFill='rbxassetid://118783818997054',laptopcomputer=
'rbxassetid://93924665405931',laptopcomputerAndArrowDown=
'rbxassetid://136939371714748',laptopcomputerBadgeCheckmark=
'rbxassetid://120482812026626',laptopcomputerSlash='rbxassetid://79424129365987'
,laptopcomputerTrianglebadgeExclamationmark='rbxassetid://120513527990773',
larisign='rbxassetid://136485640218866',
larisignArrowTriangleheadCounterclockwiseRotate90='rbxassetid://121246946230961'
,larisignBankBuilding='rbxassetid://111789107676481',larisignBankBuildingFill=
'rbxassetid://134210738755691',larisignCircle='rbxassetid://92080196175408',
larisignCircleFill='rbxassetid://109430195650783',
larisignGaugeChartLefthalfRighthalf='rbxassetid://104420126879803',
larisignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://112845157907157',
larisignRing='rbxassetid://82146466164245',larisignRingDashed=
'rbxassetid://137685798255874',larisignSquare='rbxassetid://85653556559155',
larisignSquareFill='rbxassetid://89054689413231',laserBurst=
'rbxassetid://117110789250652',lasso='rbxassetid://73650965371496',
lassoBadgeSparkles='rbxassetid://100331357189805',latch2Case=
'rbxassetid://127322944927667',latch2CaseFill='rbxassetid://137373973211444',
laurelLeading='rbxassetid://74230370864247',laurelLeadingLaurelTrailing=
'rbxassetid://121289783441119',laurelTrailing='rbxassetid://100164466613687',
lbButtonRoundedbottomHorizontal='rbxassetid://88293514053614',
lbButtonRoundedbottomHorizontalFill='rbxassetid://98021114053942',lbCircle=
'rbxassetid://74325404659334',lbCircleFill='rbxassetid://71487768652535',leaf=
'rbxassetid://137721714285432',leafArrowTriangleheadClockwise=
'rbxassetid://113114088100330',leafCircle='rbxassetid://86174952690199',
leafCircleFill='rbxassetid://93634187189693',leafFill=
'rbxassetid://113484795763411',left='rbxassetid://107416334182822',leftCircle=
'rbxassetid://119934421439703',leftCircleFill='rbxassetid://105463146320094',
lessthan='rbxassetid://94199123453074',lessthanCircle=
'rbxassetid://111066641037520',lessthanCircleFill='rbxassetid://100079024283083'
,lessthanSquare='rbxassetid://84887643197832',lessthanSquareFill=
'rbxassetid://113303675678039',lessthanorequalto='rbxassetid://90520186962478',
lessthanorequaltoCircle='rbxassetid://89396258122438',
lessthanorequaltoCircleFill='rbxassetid://80241893107159',
lessthanorequaltoSquare='rbxassetid://103756445378013',
lessthanorequaltoSquareFill='rbxassetid://113570432934713',level=
'rbxassetid://75072843777421',levelFill='rbxassetid://128257992127602',
licenseplate='rbxassetid://130402934113130',licenseplateFill=
'rbxassetid://117121209379254',lifepreserver='rbxassetid://125826954375015',
lifepreserverFill='rbxassetid://118868969071769',lightBeaconMax=
'rbxassetid://75066576103161',lightBeaconMaxFill='rbxassetid://87530931992175',
lightBeaconMin='rbxassetid://107757740522836',lightBeaconMinFill=
'rbxassetid://100129869918387',lightCylindricalCeiling=
'rbxassetid://110334573858912',lightCylindricalCeilingFill=
'rbxassetid://104211467198230',lightCylindricalCeilingInverse=
'rbxassetid://75728133446100',lightMax='rbxassetid://84057115606407',lightMin=
'rbxassetid://125533100436531',lightOverheadLeft='rbxassetid://122835138684644',
lightOverheadLeftFill='rbxassetid://111580070717708',lightOverheadRight=
'rbxassetid://117111693005736',lightOverheadRightFill=
'rbxassetid://98686619044486',lightPanel='rbxassetid://106777073156176',
lightPanelFill='rbxassetid://110860563202036',lightRecessed=
'rbxassetid://126364480423039',lightRecessed3='rbxassetid://130398146961010',
lightRecessed3Fill='rbxassetid://104115923130457',lightRecessed3Inverse=
'rbxassetid://103615064195110',lightRecessedFill='rbxassetid://88775166873026',
lightRecessedInverse='rbxassetid://78252492817119',lightRibbon=
'rbxassetid://122142963253357',lightRibbonFill='rbxassetid://93066238961423',
lightStrip2='rbxassetid://134019470579747',lightStrip2Fill=
'rbxassetid://127276872174766',lightbulb='rbxassetid://82644985572724',
lightbulb2='rbxassetid://112234784324905',lightbulb2Fill=
'rbxassetid://127418974043306',lightbulbCircle='rbxassetid://126479516799358',
lightbulbCircleFill='rbxassetid://120553924518927',lightbulbFill=
'rbxassetid://73118667835537',lightbulbLed='rbxassetid://86111405001092',
lightbulbLedFill='rbxassetid://114870437439505',lightbulbLedWide=
'rbxassetid://117190619134004',lightbulbLedWideFill=
'rbxassetid://102938343676175',lightbulbMax='rbxassetid://122168294554782',
lightbulbMaxFill='rbxassetid://90138003500408',lightbulbMin=
'rbxassetid://83516231106615',lightbulbMinBadgeExclamationmark=
'rbxassetid://97382076149987',lightbulbMinBadgeExclamationmarkFill=
'rbxassetid://103335346024590',lightbulbMinFill='rbxassetid://70934321968116',
lightbulbSlash='rbxassetid://86800234658429',lightbulbSlashFill=
'rbxassetid://132786331318088',lightrail='rbxassetid://96129918016259',
lightrailFill='rbxassetid://76488782471583',lightspectrumHorizontal=
'rbxassetid://135984270467670',lightswitchOff='rbxassetid://80355295612879',
lightswitchOffFill='rbxassetid://128742926647255',lightswitchOffSquare=
'rbxassetid://97372717984343',lightswitchOffSquareFill=
'rbxassetid://128134911832629',lightswitchOn='rbxassetid://95538959639294',
lightswitchOnFill='rbxassetid://115337416605268',lightswitchOnSquare=
'rbxassetid://114272644374823',lightswitchOnSquareFill=
'rbxassetid://117381560381596',line2HorizontalDecreaseCircle=
'rbxassetid://117477657723597',line2HorizontalDecreaseCircleFill=
'rbxassetid://118871890045553',line3CrossedSwirlCircle=
'rbxassetid://118318609660111',line3CrossedSwirlCircleFill=
'rbxassetid://134883004157167',line3Horizontal='rbxassetid://111782926831220',
line3HorizontalButtonAngledtopVerticalRight='rbxassetid://111887877381966',
line3HorizontalButtonAngledtopVerticalRightFill='rbxassetid://105369276135548',
line3HorizontalCircle='rbxassetid://85990723384739',line3HorizontalCircleFill=
'rbxassetid://134954138715896',line3HorizontalDecrease=
'rbxassetid://88784585171582',line3HorizontalDecreaseCircle=
'rbxassetid://111870154383622',line3HorizontalDecreaseCircleFill=
'rbxassetid://111120609004176',lineDiagonal='rbxassetid://137379155901548',
lineDiagonalTriangleheadUpRight='rbxassetid://99642044971640',
lineDiagonalTriangleheadUpRightLeftDown='rbxassetid://117149190547171',
lineHorizontalStarFillLineHorizontal='rbxassetid://115363674331340',
linesMeasurementHorizontal='rbxassetid://136276260519777',
linesMeasurementHorizontalAlignedBottom='rbxassetid://83461597148439',
linesMeasurementVertical='rbxassetid://84074029327107',lineweight=
'rbxassetid://97237441591248',link='rbxassetid://136619733214436',linkBadgePlus=
'rbxassetid://131321360443285',linkCircle='rbxassetid://87090441280339',
linkCircleFill='rbxassetid://107749660504734',linkIcloud=
'rbxassetid://111707841053107',linkIcloudFill='rbxassetid://105624820343494',
lirasign='rbxassetid://84073855052381',
lirasignArrowTriangleheadCounterclockwiseRotate90='rbxassetid://112189369114653'
,lirasignBankBuilding='rbxassetid://73850619799669',lirasignBankBuildingFill=
'rbxassetid://107050841352588',lirasignCircle='rbxassetid://90679509165073',
lirasignCircleFill='rbxassetid://120896567895081',
lirasignGaugeChartLefthalfRighthalf='rbxassetid://81758283721386',
lirasignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://85227261273468',
lirasignRing='rbxassetid://107360539711488',lirasignRingDashed=
'rbxassetid://120907404527155',lirasignSquare='rbxassetid://81175836370962',
lirasignSquareFill='rbxassetid://72540101703939',listAndFilm=
'rbxassetid://124042334716524',listBullet='rbxassetid://85287233895860',
listBulletBadgeEllipsis='rbxassetid://140007423548089',listBulletBelowRectangle=
'rbxassetid://97820811515269',listBulletCircle='rbxassetid://106489365197935',
listBulletCircleFill='rbxassetid://76642065569774',listBulletClipboard=
'rbxassetid://128179771023669',listBulletClipboardFill=
'rbxassetid://126582430954250',listBulletIndent='rbxassetid://98732304151282',
listBulletRectangle='rbxassetid://92743956457723',listBulletRectangleFill=
'rbxassetid://133554938646728',listBulletRectanglePortrait=
'rbxassetid://92563978202451',listBulletRectanglePortraitFill=
'rbxassetid://85234388613283',listClipboard='rbxassetid://75088463417517',
listClipboardFill='rbxassetid://113932200461935',listDash=
'rbxassetid://100354325683976',listDashBadgeEllipsis=
'rbxassetid://84510667031315',listDashHeaderRectangle=
'rbxassetid://76123478093568',listDashHeaderRectangleFill=
'rbxassetid://137696701087503',listNumber='rbxassetid://125517805939041',
listNumberBadgeEllipsis='rbxassetid://106543491819062',listStar=
'rbxassetid://88814060523780',listTriangle='rbxassetid://137615361230392',
livephoto='rbxassetid://101779027126388',livephotoBadgeAutomatic=
'rbxassetid://98509238080247',livephotoPlay='rbxassetid://76487982956241',
livephotoSlash='rbxassetid://88783391904094',lizard=
'rbxassetid://81394090984711',lizardCircle='rbxassetid://131956914242763',
lizardCircleFill='rbxassetid://89102359342570',lizardFill=
'rbxassetid://129667686919250',lmButtonHorizontal='rbxassetid://124518663294778'
,lmButtonHorizontalFill='rbxassetid://137794225984963',location=
'rbxassetid://87454675226726',locationApp='rbxassetid://108285658100441',
locationAppFill='rbxassetid://115997390547074',locationCircle=
'rbxassetid://102031654429688',locationCircleFill='rbxassetid://129255026559329'
,locationFill='rbxassetid://104193509962276',locationFillViewfinder=
'rbxassetid://72520718163831',locationMagnifyingglass=
'rbxassetid://130738172515937',locationNorth='rbxassetid://91961895110874',
locationNorthCircle='rbxassetid://92630833006450',locationNorthCircleFill=
'rbxassetid://115261018724721',locationNorthFill='rbxassetid://82896630625065',
locationNorthLine='rbxassetid://124393707860080',locationNorthLineFill=
'rbxassetid://88364457142872',locationSlash='rbxassetid://91986292380461',
locationSlashCircle='rbxassetid://133057661198672',locationSlashCircleFill=
'rbxassetid://120666508274749',locationSlashFill='rbxassetid://74991633260544',
locationSquare='rbxassetid://139433662072438',locationSquareFill=
'rbxassetid://103262354171629',locationViewfinder='rbxassetid://107082380893473'
,lock='rbxassetid://75059666276795',lockAppDashed='rbxassetid://73622959173423',
lockApplewatch='rbxassetid://137701880303372',lockBadgeCheckmark=
'rbxassetid://96107980975323',lockBadgeCheckmarkFill=
'rbxassetid://125253362602913',lockBadgeClock='rbxassetid://125387425242561',
lockBadgeClockFill='rbxassetid://103943585601256',lockBadgeXmark=
'rbxassetid://89080108534797',lockBadgeXmarkFill='rbxassetid://128829841123231',
lockCircle='rbxassetid://104937122562252',lockCircleDotted=
'rbxassetid://129061462044090',lockCircleFill='rbxassetid://88199476099135',
lockDesktopcomputer='rbxassetid://123694402953182',lockDisplay=
'rbxassetid://107733113122735',lockDocument='rbxassetid://136441673712336',
lockDocumentFill='rbxassetid://80722327719292',lockFill=
'rbxassetid://113641566747489',lockHeart='rbxassetid://112515510772540',
lockHeartFill='rbxassetid://100353400294809',lockIcloud=
'rbxassetid://115234882168637',lockIcloudFill='rbxassetid://90477192363540',
lockIpad='rbxassetid://73607136773648',lockIphone='rbxassetid://102130527102632'
,lockLaptopcomputer='rbxassetid://128883775448444',lockOpen=
'rbxassetid://102783678054610',lockOpenApplewatch='rbxassetid://128492872401861'
,lockOpenDesktopcomputer='rbxassetid://127601575525114',lockOpenDisplay=
'rbxassetid://117843429515636',lockOpenFill='rbxassetid://126165516505365',
lockOpenIpad='rbxassetid://115780599199820',lockOpenIphone=
'rbxassetid://102480932156704',lockOpenLaptopcomputer=
'rbxassetid://74819525049953',lockOpenRotation='rbxassetid://109151898406267',
lockOpenTrianglebadgeExclamationmark='rbxassetid://103088164308480',
lockOpenTrianglebadgeExclamationmarkFill='rbxassetid://130688357894527',
lockRectangle='rbxassetid://140203888025279',lockRectangleDashed=
'rbxassetid://92222544678056',lockRectangleFill='rbxassetid://96253959861246',
lockRectangleOnRectangle='rbxassetid://119424922741745',
lockRectangleOnRectangleDashed='rbxassetid://89856654852797',
lockRectangleOnRectangleFill='rbxassetid://107606810987041',lockRectangleStack=
'rbxassetid://83653436779481',lockRectangleStackFill=
'rbxassetid://114142898348233',lockRotation='rbxassetid://94299651317583',
lockShield='rbxassetid://117164861385150',lockShieldFill=
'rbxassetid://103824281733541',lockSlash='rbxassetid://109892378159236',
lockSlashFill='rbxassetid://135056397155042',lockSquare=
'rbxassetid://100512753136328',lockSquareDashed='rbxassetid://95009632896829',
lockSquareFill='rbxassetid://131560134146883',lockSquareStack=
'rbxassetid://83170488776527',lockSquareStackFill='rbxassetid://83676185405981',
lockTrianglebadgeExclamationmark='rbxassetid://130478937095272',
lockTrianglebadgeExclamationmarkFill='rbxassetid://106705811514881',
longTextPageAndPencil='rbxassetid://129681886055539',longTextPageAndPencilFill=
'rbxassetid://110646299611764',loupe='rbxassetid://136500905165325',
lsbButtonAngledbottomHorizontalLeft='rbxassetid://97637835138960',
lsbButtonAngledbottomHorizontalLeftFill='rbxassetid://136029832054596',
ltButtonRoundedtopHorizontal='rbxassetid://96312958043926',
ltButtonRoundedtopHorizontalFill='rbxassetid://127843467436825',ltCircle=
'rbxassetid://132916592513600',ltCircleFill='rbxassetid://108356683738396',lungs
='rbxassetid://117606081876003',lungsFill='rbxassetid://140685394378840',
m1ButtonHorizontal='rbxassetid://117446058381458',m1ButtonHorizontalFill=
'rbxassetid://136281737177063',m2ButtonHorizontal='rbxassetid://119180376696999'
,m2ButtonHorizontalFill='rbxassetid://91002798049706',m3ButtonHorizontal=
'rbxassetid://113743093040871',m3ButtonHorizontalFill=
'rbxassetid://115674075172130',m4ButtonHorizontal='rbxassetid://81579669296004',
m4ButtonHorizontalFill='rbxassetid://107318562519231',mCircle=
'rbxassetid://116476710361493',mCircleFill='rbxassetid://89150182712210',mSquare
='rbxassetid://105213421328907',mSquareFill='rbxassetid://100315327640869',
macbook='rbxassetid://107619424561359',macbookAndApplewatch=
'rbxassetid://134730538343771',macbookAndIpad='rbxassetid://84294510781873',
macbookAndIphone='rbxassetid://113315563711474',macbookAndIpod=
'rbxassetid://104381697175744',macbookAndVisionPro=
'rbxassetid://129796583480947',macbookBadgeCheckmark=
'rbxassetid://137440139904870',macbookBadgeExclamationmark=
'rbxassetid://85004643818160',macbookBadgeShieldCheckmark=
'rbxassetid://88995389590679',macbookGen1='rbxassetid://74836101201182',
macbookGen1Sizes='rbxassetid://112015951125344',macbookGen2=
'rbxassetid://112929584796976',macbookGen2Sizes='rbxassetid://72432237281471',
macbookSizes='rbxassetid://107797400966607',macbookSlash=
'rbxassetid://88363794341187',macbookTrianglebadgeExclamationmark=
'rbxassetid://98986359944726',macmini='rbxassetid://125932491030705',
macminiBadgeCheckmark='rbxassetid://124390002270280',macminiBadgeCheckmarkFill=
'rbxassetid://134379510662161',macminiFill='rbxassetid://111771453982507',
macminiGen2='rbxassetid://124604325683137',macminiGen2Fill=
'rbxassetid://138242624341969',macminiGen3='rbxassetid://136203836384692',
macminiGen3Fill='rbxassetid://108766825519262',macproGen1=
'rbxassetid://113873630917668',macproGen1Fill='rbxassetid://123589284967809',
macproGen2='rbxassetid://94400511740550',macproGen2Fill=
'rbxassetid://120147027066863',macproGen3='rbxassetid://89147890594992',
macproGen3BadgeCkeckmark='rbxassetid://106301224248574',
macproGen3BadgeCkeckmarkFill='rbxassetid://84360535979337',macproGen3Fill=
'rbxassetid://129039223637475',macproGen3Server='rbxassetid://108932834322099',
macstudio='rbxassetid://140406615283661',macstudioBadgeCheckmark=
'rbxassetid://77043299238345',macstudioBadgeCheckmarkFill=
'rbxassetid://121308069902452',macstudioFill='rbxassetid://140537816153199',
macwindow='rbxassetid://120538366729599',macwindowAndPointerArrow=
'rbxassetid://106055489705570',macwindowBadgePlus='rbxassetid://89261664954280',
macwindowOnRectangle='rbxassetid://72758136163509',macwindowStack=
'rbxassetid://134609194698391',magazine='rbxassetid://88500461277714',
magazineFill='rbxassetid://91312620388624',magicmouse=
'rbxassetid://92566698707027',magicmouseFill='rbxassetid://116129840309244',
magnifyingglass='rbxassetid://91129038063259',magnifyingglassCircle=
'rbxassetid://71402731578576',magnifyingglassCircleFill=
'rbxassetid://98512634782850',magsafeBatterypack='rbxassetid://131576271474358',
magsafeBatterypackFill='rbxassetid://93439484481400',mail=
'rbxassetid://135177457181908',mailAndTextMagnifyingglass=
'rbxassetid://86308377112063',mailFill='rbxassetid://97466777240439',mailStack=
'rbxassetid://113856245093303',mailStackFill='rbxassetid://93335421635768',
malaysianringgitsign='rbxassetid://122013191520543',
malaysianringgitsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://95745537119203',malaysianringgitsignBankBuilding=
'rbxassetid://95567815149875',malaysianringgitsignBankBuildingFill=
'rbxassetid://73016241207079',malaysianringgitsignCircle=
'rbxassetid://86469703884694',malaysianringgitsignCircleFill=
'rbxassetid://75744193587657',malaysianringgitsignGaugeChartLefthalfRighthalf=
'rbxassetid://123440228784595',
malaysianringgitsignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://96465358645429',malaysianringgitsignRing=
'rbxassetid://133834328268893',malaysianringgitsignRingDashed=
'rbxassetid://90883710998179',malaysianringgitsignSquare=
'rbxassetid://85460771757387',malaysianringgitsignSquareFill=
'rbxassetid://107166651287050',manatsign='rbxassetid://139143453917100',
manatsignArrowTriangleheadCounterclockwiseRotate90='rbxassetid://70744179468784'
,manatsignBankBuilding='rbxassetid://76245262473506',manatsignBankBuildingFill=
'rbxassetid://114034267948261',manatsignCircle='rbxassetid://126199967859257',
manatsignCircleFill='rbxassetid://140593155589193',
manatsignGaugeChartLefthalfRighthalf='rbxassetid://118055801527619',
manatsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://82336814813784',
manatsignRing='rbxassetid://109963598001949',manatsignRingDashed=
'rbxassetid://81986929118943',manatsignSquare='rbxassetid://73346293183441',
manatsignSquareFill='rbxassetid://99779949454450',map=
'rbxassetid://93531590659266',mapCircle='rbxassetid://134601191431751',
mapCircleFill='rbxassetid://106092734589750',mapFill=
'rbxassetid://132139046895902',mappin='rbxassetid://121615146959714',
mappinAndEllipse='rbxassetid://121230726923110',mappinAndEllipseCircle=
'rbxassetid://125429828094681',mappinAndEllipseCircleFill=
'rbxassetid://139128577846173',mappinCircle='rbxassetid://92261727453112',
mappinCircleFill='rbxassetid://92260033202215',mappinSlash=
'rbxassetid://132771607889983',mappinSlashCircle='rbxassetid://74525734290852',
mappinSlashCircleFill='rbxassetid://119484802415308',mappinSquare=
'rbxassetid://104935304695180',mappinSquareFill='rbxassetid://86746492573973',
matterLogo='rbxassetid://102406572400014',mecca='rbxassetid://127812717762114',
medal='rbxassetid://71033385043976',medalFill='rbxassetid://90953143843307',
medalStar='rbxassetid://120019164471743',medalStarFill=
'rbxassetid://80020007851214',mediastick='rbxassetid://109549165782172',
medicalThermometer='rbxassetid://136280158502496',medicalThermometerFill=
'rbxassetid://119779476439580',megaphone='rbxassetid://117018716409794',
megaphoneFill='rbxassetid://138594117290847',memories=
'rbxassetid://81385055592525',memoriesBadgeCheckmark=
'rbxassetid://101694983725997',memoriesBadgeMinus='rbxassetid://85572709182032',
memoriesBadgePlus='rbxassetid://89625059340245',memoriesBadgeXmark=
'rbxassetid://119611626635819',memoriesSlash='rbxassetid://119477438102752',
memorychip='rbxassetid://72442769456866',memorychipFill=
'rbxassetid://108024909200103',menubarArrowDownRectangle=
'rbxassetid://78264857291724',menubarArrowUpRectangle=
'rbxassetid://79499600876886',menubarDockRectangle='rbxassetid://73630204597242'
,menubarDockRectangleBadgeRecord='rbxassetid://70512793203960',menubarRectangle=
'rbxassetid://138854721784790',menucard='rbxassetid://140366101750659',
menucardFill='rbxassetid://124582943019876',message=
'rbxassetid://74395662017461',messageBadge='rbxassetid://100483745924724',
messageBadgeCircle='rbxassetid://106085199916148',messageBadgeCircleFill=
'rbxassetid://102395002843600',messageBadgeFill='rbxassetid://92248945524593',
messageBadgeFilledFill='rbxassetid://89587973073375',messageBadgeWaveform=
'rbxassetid://83330938459388',messageBadgeWaveformFill=
'rbxassetid://130089782798576',messageCircle='rbxassetid://102977548422597',
messageCircleFill='rbxassetid://98654060282404',messageFill=
'rbxassetid://78470145456613',metronome='rbxassetid://106160116449584',
metronomeFill='rbxassetid://113113603878861',microbe=
'rbxassetid://101055634388755',microbeCircle='rbxassetid://71746458377734',
microbeCircleFill='rbxassetid://102842672445570',microbeFill=
'rbxassetid://89507986671859',microphone='rbxassetid://91483706052452',
microphoneAndSignalMeter='rbxassetid://92063694320161',
microphoneAndSignalMeterFill='rbxassetid://106138035371214',
microphoneBadgeEllipsis='rbxassetid://103989002655467',
microphoneBadgeEllipsisFill='rbxassetid://110996273867879',microphoneBadgePlus=
'rbxassetid://117094983790077',microphoneBadgePlusFill=
'rbxassetid://127145588822404',microphoneBadgeXmark=
'rbxassetid://74319653306618',microphoneBadgeXmarkFill=
'rbxassetid://72163348631659',microphoneCircle='rbxassetid://128457079809321',
microphoneCircleFill='rbxassetid://124876886582740',microphoneFill=
'rbxassetid://93919436903011',microphoneSlash='rbxassetid://128089521766130',
microphoneSlashCircle='rbxassetid://81904243939624',microphoneSlashCircleFill=
'rbxassetid://85089208509744',microphoneSlashFill='rbxassetid://114274894150098'
,microphoneSquare='rbxassetid://114042644452492',microphoneSquareFill=
'rbxassetid://100514210081121',microwave='rbxassetid://109825403791681',
microwaveFill='rbxassetid://72225124273007',millsign=
'rbxassetid://120612264843894',millsignArrowTriangleheadCounterclockwiseRotate90
='rbxassetid://113401890675382',millsignBankBuilding=
'rbxassetid://84067991643769',millsignBankBuildingFill=
'rbxassetid://137124816253943',millsignCircle='rbxassetid://93145949048334',
millsignCircleFill='rbxassetid://119498566433333',
millsignGaugeChartLefthalfRighthalf='rbxassetid://87724227677679',
millsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://136203853226981',
millsignRing='rbxassetid://116690436863629',millsignRingDashed=
'rbxassetid://110814114561866',millsignSquare='rbxassetid://77492773148562',
millsignSquareFill='rbxassetid://106288582641608',minus=
'rbxassetid://89004332408449',minusArrowTriangleheadClockwise=
'rbxassetid://114634757784622',minusArrowTriangleheadCounterclockwise=
'rbxassetid://107045388378537',minusCircle='rbxassetid://137503047808593',
minusCircleFill='rbxassetid://114233698275838',minusDiamond=
'rbxassetid://70626236941066',minusDiamondFill='rbxassetid://137055603164475',
minusForwardslashPlus='rbxassetid://139481511724753',minusMagnifyingglass=
'rbxassetid://134132342538161',minusPlusAndFluidBatteryblock=
'rbxassetid://101939616357249',minusPlusBatteryblock=
'rbxassetid://127012882466494',minusPlusBatteryblockExclamationmark=
'rbxassetid://139500256583872',minusPlusBatteryblockExclamationmarkFill=
'rbxassetid://96081928780644',minusPlusBatteryblockFill=
'rbxassetid://117085342503922',minusPlusBatteryblockSlash=
'rbxassetid://100393676834035',minusPlusBatteryblockSlashFill=
'rbxassetid://124640512084539',minusPlusBatteryblockStack=
'rbxassetid://79524287782222',minusPlusBatteryblockStackArrowtriangleLeft=
'rbxassetid://122261351206637',minusPlusBatteryblockStackArrowtriangleLeftFill=
'rbxassetid://115291043746629',minusPlusBatteryblockStackArrowtriangleRight=
'rbxassetid://91966942937693',
minusPlusBatteryblockStackArrowtriangleRightAndArrowtriangleLeft=
'rbxassetid://134386149198478',
minusPlusBatteryblockStackArrowtriangleRightAndArrowtriangleLeftFill=
'rbxassetid://88265395694017',minusPlusBatteryblockStackArrowtriangleRightFill=
'rbxassetid://121219514043882',minusPlusBatteryblockStackExclamationmark=
'rbxassetid://123187360686442',minusPlusBatteryblockStackExclamationmarkFill=
'rbxassetid://72073968796558',minusPlusBatteryblockStackFill=
'rbxassetid://129390097165145',minusPlusLinesMeasurementHorizontalAlignedBottom=
'rbxassetid://86287457532753',minusRectangle='rbxassetid://101832378845362',
minusRectangleFill='rbxassetid://71255897440595',minusRectanglePortrait=
'rbxassetid://79554895569395',minusRectanglePortraitFill=
'rbxassetid://123355657429657',minusSquare='rbxassetid://92869320034967',
minusSquareFill='rbxassetid://93926501938010',mirrorSideLeft=
'rbxassetid://79676503129954',mirrorSideLeftAndArrowTurnDownRight=
'rbxassetid://112735732212436',mirrorSideLeftAndHeatWaves=
'rbxassetid://97452704291102',mirrorSideRight='rbxassetid://83056491035621',
mirrorSideRightAndArrowTurnDownLeft='rbxassetid://118420715252545',
mirrorSideRightAndHeatWaves='rbxassetid://74725135702403',moon=
'rbxassetid://82139899083256',moonCircle='rbxassetid://85414026848067',
moonCircleFill='rbxassetid://127455736032544',moonDust=
'rbxassetid://80023875298780',moonDustCircle='rbxassetid://137917787774386',
moonDustCircleFill='rbxassetid://89884876621538',moonDustFill=
'rbxassetid://92755134821567',moonFill='rbxassetid://80943018865798',moonHaze=
'rbxassetid://127865614718648',moonHazeCircle='rbxassetid://104234616092079',
moonHazeCircleFill='rbxassetid://72550900291532',moonHazeFill=
'rbxassetid://132090014257174',moonRoadLanes='rbxassetid://116463811762109',
moonStars='rbxassetid://87321082119205',moonStarsCircle=
'rbxassetid://75028167300841',moonStarsCircleFill='rbxassetid://83056445144978',
moonStarsFill='rbxassetid://117208258351328',moonZzz=
'rbxassetid://90271070534923',moonZzzFill='rbxassetid://120704152209521',
moonphaseFirstQuarter='rbxassetid://117116577141627',
moonphaseFirstQuarterInverse='rbxassetid://111603319059305',moonphaseFullMoon=
'rbxassetid://101123767009739',moonphaseFullMoonInverse=
'rbxassetid://87181814547523',moonphaseLastQuarter=
'rbxassetid://125883223406968',moonphaseLastQuarterInverse=
'rbxassetid://113591277926901',moonphaseNewMoon='rbxassetid://91400655736524',
moonphaseNewMoonInverse='rbxassetid://94223725414952',moonphaseWaningCrescent=
'rbxassetid://71889864039420',moonphaseWaningCrescentInverse=
'rbxassetid://97971828509046',moonphaseWaningGibbous=
'rbxassetid://125561957914316',moonphaseWaningGibbousInverse=
'rbxassetid://133040082683778',moonphaseWaxingCrescent=
'rbxassetid://109118424981772',moonphaseWaxingCrescentInverse=
'rbxassetid://83389541865251',moonphaseWaxingGibbous=
'rbxassetid://117103885736080',moonphaseWaxingGibbousInverse=
'rbxassetid://81404317227610',moonrise='rbxassetid://115339490944432',
moonriseCircle='rbxassetid://136045524355967',moonriseCircleFill=
'rbxassetid://128404111997844',moonriseFill='rbxassetid://103140476284194',
moonset='rbxassetid://72866577933832',moonsetCircle=
'rbxassetid://74071623128842',moonsetCircleFill='rbxassetid://132376217303310',
moonsetFill='rbxassetid://82467208130648',moped='rbxassetid://132510221978497',
mopedFill='rbxassetid://118529287468863',mosaic='rbxassetid://89243776028649',
mosaicFill='rbxassetid://95720390375780',motorcycle=
'rbxassetid://82758135256909',motorcycleFill='rbxassetid://129675811040460',
mount='rbxassetid://118225973874297',mountFill='rbxassetid://136416370964291',
mountain2='rbxassetid://73789953244069',mountain2Circle=
'rbxassetid://120539292840338',mountain2CircleFill=
'rbxassetid://126616357820763',mountain2Fill='rbxassetid://72619625972809',mouth
='rbxassetid://72658267669640',mouthFill='rbxassetid://100047036743743',move3d=
'rbxassetid://112398672404328',movieclapper='rbxassetid://80954194772844',
movieclapperFill='rbxassetid://135561411591078',mph=
'rbxassetid://128529074792406',mphCircle='rbxassetid://81809274837519',
mphCircleFill='rbxassetid://121437826766049',mug='rbxassetid://110628058332506',
mugFill='rbxassetid://139140377653653',multiply='rbxassetid://136891818007188',
multiplyCircle='rbxassetid://127978128794846',multiplyCircleFill=
'rbxassetid://127229291230155',multiplySquare='rbxassetid://97309344874453',
multiplySquareFill='rbxassetid://112070371616255',musicMicrophone=
'rbxassetid://100824169660656',musicMicrophoneCircle=
'rbxassetid://131498749144552',musicMicrophoneCircleFill=
'rbxassetid://120791444280471',musicNote='rbxassetid://126082822807584',
musicNoteArrowTriangleheadClockwise='rbxassetid://133288353146842',
musicNoteHouse='rbxassetid://91704654202540',musicNoteHouseFill=
'rbxassetid://99180255941950',musicNoteList='rbxassetid://96230339803471',
musicNoteSlash='rbxassetid://127771624449291',musicNoteSquareStack=
'rbxassetid://88198486537881',musicNoteSquareStackFill=
'rbxassetid://136399162566805',musicNoteTv='rbxassetid://118014044046358',
musicNoteTvFill='rbxassetid://101072939696766',musicPages=
'rbxassetid://98131195358883',musicPagesFill='rbxassetid://100815742404062',
musicQuarternote3='rbxassetid://72571169895421',mustache=
'rbxassetid://81213625247949',mustacheFill='rbxassetid://135889093221272',
nCircle='rbxassetid://119692721835646',nCircleFill=
'rbxassetid://139590129950612',nSquare='rbxassetid://82639756767208',nSquareFill
='rbxassetid://77756021605509',nairasign='rbxassetid://106795085125512',
nairasignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://136163098752557',nairasignBankBuilding=
'rbxassetid://118269030455068',nairasignBankBuildingFill=
'rbxassetid://102921613899751',nairasignCircle='rbxassetid://75558341136080',
nairasignCircleFill='rbxassetid://74947644324829',
nairasignGaugeChartLefthalfRighthalf='rbxassetid://129556356646485',
nairasignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://96821649386281',
nairasignRing='rbxassetid://117523908784311',nairasignRingDashed=
'rbxassetid://105742853876513',nairasignSquare='rbxassetid://135120380049763',
nairasignSquareFill='rbxassetid://132052855718549',network=
'rbxassetid://120498202241355',networkBadgeShieldHalfFilled=
'rbxassetid://113813057412690',networkSlash='rbxassetid://101003667032615',
newspaper='rbxassetid://104839010601075',newspaperCircle=
'rbxassetid://115347908928306',newspaperCircleFill=
'rbxassetid://105464060542335',newspaperFill='rbxassetid://137360183744130',
norwegiankronesign='rbxassetid://119731208635088',
norwegiankronesignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://101411001215020',norwegiankronesignBankBuilding=
'rbxassetid://85695886048676',norwegiankronesignBankBuildingFill=
'rbxassetid://95684757727036',norwegiankronesignCircle=
'rbxassetid://81846373779456',norwegiankronesignCircleFill=
'rbxassetid://109254117252040',norwegiankronesignGaugeChartLefthalfRighthalf=
'rbxassetid://113781025133033',
norwegiankronesignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://78092984701784',norwegiankronesignRing=
'rbxassetid://138947979267546',norwegiankronesignRingDashed=
'rbxassetid://77241946000543',norwegiankronesignSquare=
'rbxassetid://95995824513915',norwegiankronesignSquareFill=
'rbxassetid://96123508833499',nose='rbxassetid://74544706026614',noseFill=
'rbxassetid://124371438722659',nosign='rbxassetid://110675356214733',nosignApp=
'rbxassetid://90400743739520',nosignAppFill='rbxassetid://107658749612961',
nosignBadgeClock='rbxassetid://137939836191842',notequal=
'rbxassetid://86562600167503',notequalCircle='rbxassetid://135789760267660',
notequalCircleFill='rbxassetid://75312888708455',notequalSquare=
'rbxassetid://81897503982741',notequalSquareFill='rbxassetid://72731357908400',
number='rbxassetid://127208629756456',numberCircle='rbxassetid://91537062236961'
,numberCircleFill='rbxassetid://128309928890652',numberSquare=
'rbxassetid://82329567357914',numberSquareFill='rbxassetid://126805206921861',
numbers='rbxassetid://84498045988050',numbersRectangle=
'rbxassetid://94260654225058',numbersRectangleFill='rbxassetid://94598633915174'
,numbersign='rbxassetid://95244886174960',oCircle='rbxassetid://106397024350624'
,oCircleFill='rbxassetid://76217943049253',oSquare='rbxassetid://73833935065451'
,oSquareFill='rbxassetid://96189217216385',oar2Crossed=
'rbxassetid://87527605793944',oar2CrossedCircle='rbxassetid://136436924320890',
oar2CrossedCircleFill='rbxassetid://101685289108994',octagon=
'rbxassetid://79038916131641',octagonBottomhalfFilled=
'rbxassetid://114692214649659',octagonFill='rbxassetid://107625404272807',
octagonLefthalfFilled='rbxassetid://99209930883951',octagonRighthalfFilled=
'rbxassetid://112842649283426',octagonTophalfFilled=
'rbxassetid://114065832521760',oilcan='rbxassetid://135272223692064',
oilcanAndThermometer='rbxassetid://117495873565426',oilcanAndThermometerFill=
'rbxassetid://112607162891091',oilcanFill='rbxassetid://118078317492971',
opticaldisc='rbxassetid://123902479206565',opticaldiscFill=
'rbxassetid://93483653186548',opticaldiscdrive='rbxassetid://73954391898782',
opticaldiscdriveFill='rbxassetid://134334854298944',opticid=
'rbxassetid://122071408513811',opticidFill='rbxassetid://120138153821378',option
='rbxassetid://132861598935618',oval='rbxassetid://113810873605238',
ovalBottomhalfFilled='rbxassetid://126916332045325',ovalFill=
'rbxassetid://94952343843714',ovalLefthalfFilled='rbxassetid://113389656237279',
ovalPortrait='rbxassetid://132483754313621',ovalPortraitBottomhalfFilled=
'rbxassetid://71422335054914',ovalPortraitFill='rbxassetid://79131278329666',
ovalPortraitLefthalfFilled='rbxassetid://109381437626839',
ovalPortraitRighthalfFilled='rbxassetid://111288429681074',
ovalPortraitTophalfFilled='rbxassetid://123536161343998',ovalRighthalfFilled=
'rbxassetid://104199544496623',ovalTophalfFilled='rbxassetid://101770441746408',
oven='rbxassetid://86234645623692',ovenFill='rbxassetid://83900517666474',
p1ButtonHorizontal='rbxassetid://126562898638354',p1ButtonHorizontalFill=
'rbxassetid://110710525929608',p2ButtonHorizontal='rbxassetid://77218364525379',
p2ButtonHorizontalFill='rbxassetid://133839416874548',p3ButtonHorizontal=
'rbxassetid://138792223238109',p3ButtonHorizontalFill=
'rbxassetid://106186118395145',p4ButtonHorizontal='rbxassetid://77853304593628',
p4ButtonHorizontalFill='rbxassetid://130778443695025',pCircle=
'rbxassetid://117640721311754',pCircleFill='rbxassetid://87813691082387',pSquare
='rbxassetid://107762110453705',pSquareFill='rbxassetid://96012467396262',
padHeader='rbxassetid://92020313753615',paddleshifterLeft=
'rbxassetid://126265871847369',paddleshifterLeftFill=
'rbxassetid://125673273656120',paddleshifterRight='rbxassetid://117840207563380'
,paddleshifterRightFill='rbxassetid://80336378319314',paintBucketClassic=
'rbxassetid://85152836091310',paintbrush='rbxassetid://109533740362394',
paintbrushFill='rbxassetid://99513788554382',paintbrushPointed=
'rbxassetid://127402849243896',paintbrushPointedFill=
'rbxassetid://88165121517273',paintpalette='rbxassetid://138490319699379',
paintpaletteFill='rbxassetid://131716243676081',pano=
'rbxassetid://107256596811076',panoBadgePlay='rbxassetid://105599907508172',
panoBadgePlayFill='rbxassetid://131513740947262',panoFill=
'rbxassetid://89694664506298',paperclip='rbxassetid://137882336475940',
paperclipBadgeEllipsis='rbxassetid://134936619429895',paperclipCircle=
'rbxassetid://110142955736619',paperclipCircleFill=
'rbxassetid://133602092264853',paperplane='rbxassetid://80974571487117',
paperplaneCircle='rbxassetid://137118091888490',paperplaneCircleFill=
'rbxassetid://71065495715325',paperplaneFill='rbxassetid://74965599538986',
paragraphsign='rbxassetid://94556416337049',parentheses=
'rbxassetid://130009418294104',parkinglight='rbxassetid://109225690951003',
parkinglightFill='rbxassetid://99882392916301',parkingsign=
'rbxassetid://75870889123264',parkingsignBrakesignal=
'rbxassetid://112789415747617',parkingsignBrakesignalSlash=
'rbxassetid://71858648062530',parkingsignCircle='rbxassetid://99556995799552',
parkingsignCircleFill='rbxassetid://87920284156496',
parkingsignRadiowavesDownRightOff='rbxassetid://138123234771263',
parkingsignRadiowavesLeftAndRight='rbxassetid://103933467168122',
parkingsignRadiowavesLeftAndRightSlash='rbxassetid://97814507902059',
parkingsignRadiowavesRightAndSafetycone='rbxassetid://104481497468858',
parkingsignSquare='rbxassetid://77549829356466',parkingsignSquareFill=
'rbxassetid://94937107598603',parkingsignSteeringwheel=
'rbxassetid://113877456839912',partyPopper='rbxassetid://97097770522299',
partyPopperFill='rbxassetid://126854429415327',pause=
'rbxassetid://86021151376401',pauseCircle='rbxassetid://121043050488805',
pauseCircleFill='rbxassetid://113265459725055',pauseFill=
'rbxassetid://133135421688612',pauseRectangle='rbxassetid://79118144830856',
pauseRectangleFill='rbxassetid://98884467605829',pawprint=
'rbxassetid://72805687268685',pawprintCircle='rbxassetid://82961293618699',
pawprintCircleFill='rbxassetid://133414719283799',pawprintFill=
'rbxassetid://120051601058569',pc='rbxassetid://84697360225761',peacesign=
'rbxassetid://83716327577706',pedalAccelerator='rbxassetid://107951710156857',
pedalAcceleratorFill='rbxassetid://130529659230331',pedalBrake=
'rbxassetid://127337111670528',pedalBrakeFill='rbxassetid://71467964364715',
pedalClutch='rbxassetid://73279295635737',pedalClutchFill=
'rbxassetid://103699453075905',pedestrianGateClosed=
'rbxassetid://110977038522546',pedestrianGateClosedTrianglebadgeExclamationmark=
'rbxassetid://84254696178218',pedestrianGateOpen='rbxassetid://131872346865052',
pedestrianGateOpenTrianglebadgeExclamationmark='rbxassetid://131146459812890',
pencil='rbxassetid://130705269869803',pencilAndListClipboard=
'rbxassetid://108221324478019',pencilAndOutline='rbxassetid://71370761213291',
pencilAndRuler='rbxassetid://130053969230904',pencilAndRulerFill=
'rbxassetid://135794725468905',pencilAndScribble='rbxassetid://71841329049910',
pencilCircle='rbxassetid://126091045493595',pencilCircleFill=
'rbxassetid://120665154874259',pencilLine='rbxassetid://114694675953537',
pencilSlash='rbxassetid://99813950298838',pencilTip=
'rbxassetid://72990698962317',pencilTipCropCircle='rbxassetid://116258147913478'
,pencilTipCropCircleBadgeArrowForward='rbxassetid://138569973971114',
pencilTipCropCircleBadgeArrowForwardFill='rbxassetid://80339580243434',
pencilTipCropCircleBadgeMinus='rbxassetid://137149264289620',
pencilTipCropCircleBadgeMinusFill='rbxassetid://106175372040589',
pencilTipCropCircleBadgePlus='rbxassetid://81912638436848',
pencilTipCropCircleBadgePlusFill='rbxassetid://135279218878675',
pencilTipCropCircleFill='rbxassetid://123542970215827',pentagon=
'rbxassetid://113187250562484',pentagonBottomhalfFilled=
'rbxassetid://77804606585420',pentagonFill='rbxassetid://107612785851729',
pentagonLefthalfFilled='rbxassetid://102272146625100',pentagonRighthalfFilled=
'rbxassetid://71901071200525',pentagonTophalfFilled=
'rbxassetid://124042937629059',percent='rbxassetid://127492144708581',person=
'rbxassetid://110701632373035',person2='rbxassetid://112399905717309',
person2ArrowTriangleheadCounterclockwise='rbxassetid://123475483919631',
person2Badge='rbxassetid://81421024779982',person2BadgeFill=
'rbxassetid://138318005759141',person2BadgeGearshape=
'rbxassetid://132573815056374',person2BadgeGearshapeFill=
'rbxassetid://73944930602710',person2BadgeKey='rbxassetid://93647666096573',
person2BadgeKeyFill='rbxassetid://136470113852487',person2BadgeMinus=
'rbxassetid://84645992076966',person2BadgeMinusFill=
'rbxassetid://77898589792614',person2BadgePlus='rbxassetid://135162242901966',
person2BadgePlusFill='rbxassetid://123701745548055',person2Circle=
'rbxassetid://91806733116755',person2CircleFill='rbxassetid://130770498958716',
person2CropSquareStack='rbxassetid://116331579710856',person2CropSquareStackFill
='rbxassetid://132329087604078',person2Fill='rbxassetid://140041117183015',
person2Shield='rbxassetid://124016438026144',person2ShieldFill=
'rbxassetid://139070659357420',person2Slash='rbxassetid://87644578467719',
person2SlashFill='rbxassetid://137522706511369',person2Wave2=
'rbxassetid://122652213620435',person2Wave2Fill='rbxassetid://76240613584680',
person3='rbxassetid://129473219512684',person3Fill=
'rbxassetid://105740784710191',person3Sequence='rbxassetid://107389006554677',
person3SequenceFill='rbxassetid://85888945527165',
personAndArrowLeftAndArrowRightOutward='rbxassetid://101409421769868',
personAndBackgroundDotted='rbxassetid://83784991200691',
personAndBackgroundStripedHorizontal='rbxassetid://75509951269289',
personBadgeClock='rbxassetid://97908584137499',personBadgeClockFill=
'rbxassetid://76924073855200',personBadgeKey='rbxassetid://98218511075004',
personBadgeKeyFill='rbxassetid://140402839580918',personBadgeMinus=
'rbxassetid://125565222576571',personBadgePlus='rbxassetid://115656988628904',
personBadgeShieldCheckmark='rbxassetid://83267413868149',
personBadgeShieldCheckmarkFill='rbxassetid://131283863488112',
personBadgeShieldExclamationmark='rbxassetid://84408939526129',
personBadgeShieldExclamationmarkFill='rbxassetid://87987647501843',personBubble=
'rbxassetid://133030906481167',personBubbleFill='rbxassetid://117517528643201',
personBust='rbxassetid://87620323313188',personBustCircle=
'rbxassetid://79288327836334',personBustCircleFill=
'rbxassetid://101400531456668',personBustFill='rbxassetid://111810892942700',
personCheckmarkAndXmark='rbxassetid://94288741955083',personCircle=
'rbxassetid://135997814266617',personCircleFill='rbxassetid://90118396547869',
personCropArtframe='rbxassetid://90385788840374',personCropBadgeMagnifyingglass=
'rbxassetid://97394616198512',personCropBadgeMagnifyingglassFill=
'rbxassetid://132389329009693',personCropCircle='rbxassetid://118995818358156',
personCropCircleBadge='rbxassetid://131364710689047',
personCropCircleBadgeCheckmark='rbxassetid://103344345950122',
personCropCircleBadgeClock='rbxassetid://127451044762338',
personCropCircleBadgeClockFill='rbxassetid://76718758852507',
personCropCircleBadgeEllipsis='rbxassetid://96642912386901',
personCropCircleBadgeEllipsisFill='rbxassetid://88343482632492',
personCropCircleBadgeExclamationmark='rbxassetid://139365729353052',
personCropCircleBadgeExclamationmarkFill='rbxassetid://109742792874521',
personCropCircleBadgeFill='rbxassetid://107018668861828',
personCropCircleBadgeMinus='rbxassetid://97855563994788',
personCropCircleBadgeMoon='rbxassetid://133337215391424',
personCropCircleBadgeMoonFill='rbxassetid://122077316360614',
personCropCircleBadgePlus='rbxassetid://81853344224695',
personCropCircleBadgeQuestionmark='rbxassetid://139239900008701',
personCropCircleBadgeQuestionmarkFill='rbxassetid://99804179170835',
personCropCircleBadgeXmark='rbxassetid://91489653759413',personCropCircleDashed=
'rbxassetid://82498046222304',personCropCircleDashedCircle=
'rbxassetid://71063283330455',personCropCircleDashedCircleFill=
'rbxassetid://137889244981033',personCropCircleFill=
'rbxassetid://124171421403082',personCropCircleFillBadgeCheckmark=
'rbxassetid://104885997312791',personCropCircleFillBadgeMinus=
'rbxassetid://81459239482197',personCropCircleFillBadgePlus=
'rbxassetid://79413393740430',personCropCircleFillBadgeXmark=
'rbxassetid://75776588765496',personCropRectangle='rbxassetid://80086996421928',
personCropRectangleBadgePlus='rbxassetid://102909379035898',
personCropRectangleBadgePlusFill='rbxassetid://76635295827459',
personCropRectangleFill='rbxassetid://105492427236467',personCropRectangleStack=
'rbxassetid://100303396427325',personCropRectangleStackFill=
'rbxassetid://121443905592285',personCropSquare='rbxassetid://73096364605342',
personCropSquareBadgeCamera='rbxassetid://135346755645322',
personCropSquareBadgeCameraFill='rbxassetid://92729904669185',
personCropSquareBadgeVideo='rbxassetid://78518587851789',
personCropSquareBadgeVideoFill='rbxassetid://114816625385618',
personCropSquareFill='rbxassetid://105841006579983',
personCropSquareFilledAndAtRectangle='rbxassetid://118924461813330',
personCropSquareFilledAndAtRectangleFill='rbxassetid://80381762756864',
personCropSquareOnSquareAngled='rbxassetid://85148546794958',
personCropSquareOnSquareAngledFill='rbxassetid://97667541433512',personFill=
'rbxassetid://87697331384088',personFillAndArrowLeftAndArrowRightOutward=
'rbxassetid://122634217727119',personFillBadgeMinus=
'rbxassetid://81924980008824',personFillBadgePlus='rbxassetid://134231365958570'
,personFillCheckmark='rbxassetid://103977509186960',personFillCheckmarkAndXmark=
'rbxassetid://101086334574836',personFillQuestionmark=
'rbxassetid://84568714854237',personFillTurnDown='rbxassetid://138058666704795',
personFillTurnLeft='rbxassetid://126438219715870',personFillTurnRight=
'rbxassetid://78937173520484',personFillViewfinder=
'rbxassetid://118288055558650',personFillXmark='rbxassetid://125072040338347',
personIcloud='rbxassetid://120820184523425',personIcloudFill=
'rbxassetid://133378049835063',personLineDottedPerson=
'rbxassetid://102084674613931',personLineDottedPersonFill=
'rbxassetid://135829432223513',personSlash='rbxassetid://72782294100335',
personSlashFill='rbxassetid://121955009050346',personSpatialaudio3dFill=
'rbxassetid://84698678474801',personSpatialaudioFill=
'rbxassetid://78014656341517',personSpatialaudioStereo3dFill=
'rbxassetid://97758734740805',personSpatialaudioStereoFill=
'rbxassetid://120513790757334',personTextRectangle=
'rbxassetid://128884463766600',personTextRectangleFill=
'rbxassetid://137564215419438',personTextRectangleTrianglebadgeExclamationmark=
'rbxassetid://80430192542431',
personTextRectangleTrianglebadgeExclamationmarkFill=
'rbxassetid://95425946810987',personWave2='rbxassetid://107176133016557',
personWave2Fill='rbxassetid://119313962458106',personalhotspot=
'rbxassetid://100375127059556',personalhotspotCircle=
'rbxassetid://75798505357181',personalhotspotCircleFill=
'rbxassetid://91105949987614',personalhotspotSlash=
'rbxassetid://121054134249173',perspective='rbxassetid://126209555168111',
peruviansolessign='rbxassetid://118377143493294',
peruviansolessignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://128283504294445',peruviansolessignBankBuilding=
'rbxassetid://102898750065832',peruviansolessignBankBuildingFill=
'rbxassetid://74069927107160',peruviansolessignCircle=
'rbxassetid://73491243272124',peruviansolessignCircleFill=
'rbxassetid://88303394405719',peruviansolessignGaugeChartLefthalfRighthalf=
'rbxassetid://95412355379458',
peruviansolessignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://105242621393318',peruviansolessignRing=
'rbxassetid://70902936676598',peruviansolessignRingDashed=
'rbxassetid://73633708956892',peruviansolessignSquare=
'rbxassetid://73308117343401',peruviansolessignSquareFill=
'rbxassetid://102892314110416',pesetasign='rbxassetid://85532413742774',
pesetasignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://95814949182701',pesetasignBankBuilding=
'rbxassetid://70435245510011',pesetasignBankBuildingFill=
'rbxassetid://76828425934083',pesetasignCircle='rbxassetid://73689448530780',
pesetasignCircleFill='rbxassetid://129145068296499',
pesetasignGaugeChartLefthalfRighthalf='rbxassetid://135650251052406',
pesetasignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://125057752565087',
pesetasignRing='rbxassetid://138901454310015',pesetasignRingDashed=
'rbxassetid://139127402491166',pesetasignSquare='rbxassetid://120082289628623',
pesetasignSquareFill='rbxassetid://92637657922110',pesosign=
'rbxassetid://131506373872953',pesosignArrowTriangleheadCounterclockwiseRotate90
='rbxassetid://95176369235434',pesosignBankBuilding=
'rbxassetid://133065437589460',pesosignBankBuildingFill=
'rbxassetid://133018359221072',pesosignCircle='rbxassetid://82936186087157',
pesosignCircleFill='rbxassetid://122355951938355',
pesosignGaugeChartLefthalfRighthalf='rbxassetid://71731520831769',
pesosignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://79927907535836',
pesosignRing='rbxassetid://96568771428940',pesosignRingDashed=
'rbxassetid://114760367933760',pesosignSquare='rbxassetid://99221907869730',
pesosignSquareFill='rbxassetid://104090017006574',petCarrier=
'rbxassetid://92747235907572',petCarrierCircle='rbxassetid://89684260845982',
petCarrierCircleFill='rbxassetid://136795292748871',petCarrierFill=
'rbxassetid://118526244393497',phone='rbxassetid://130274090404678',
phoneArrowDownLeft='rbxassetid://132462917948039',phoneArrowDownLeftFill=
'rbxassetid://87805128798557',phoneArrowRight='rbxassetid://72030777597334',
phoneArrowRightFill='rbxassetid://121404647674426',phoneArrowUpRight=
'rbxassetid://79872298851136',phoneArrowUpRightCircle=
'rbxassetid://70948578186592',phoneArrowUpRightCircleFill=
'rbxassetid://84134830279781',phoneArrowUpRightFill=
'rbxassetid://133725301955212',phoneBadgeCheckmark=
'rbxassetid://133676619626440',phoneBadgeClock='rbxassetid://105195846197897',
phoneBadgeClockFill='rbxassetid://76558752952090',phoneBadgePlus=
'rbxassetid://77530999977499',phoneBadgeWaveform='rbxassetid://82744515131009',
phoneBadgeWaveformFill='rbxassetid://115162205181132',phoneBubble=
'rbxassetid://107177706753542',phoneBubbleFill='rbxassetid://116143288406977',
phoneCircle='rbxassetid://99640263570255',phoneCircleFill=
'rbxassetid://111534598395128',phoneConnection='rbxassetid://107182975371124',
phoneConnectionFill='rbxassetid://132948841358108',phoneDown=
'rbxassetid://102456970457868',phoneDownCircle='rbxassetid://82910023145953',
phoneDownCircleFill='rbxassetid://72377511837884',phoneDownFill=
'rbxassetid://86552310564370',phoneDownWavesLeftAndRight=
'rbxassetid://136971996615575',phoneFill='rbxassetid://120429135463665',
phoneFillBadgeCheckmark='rbxassetid://115261691430286',phoneFillBadgePlus=
'rbxassetid://87819134346668',phonePause='rbxassetid://134150100031688',
phonePauseCircle='rbxassetid://110785714836445',phonePauseCircleFill=
'rbxassetid://78369532938975',phonePauseFill='rbxassetid://95822352892272',photo
='rbxassetid://132914683561588',photoArtframe='rbxassetid://87277781880011',
photoArtframeCircle='rbxassetid://130558529142621',photoArtframeCircleFill=
'rbxassetid://137928811461182',photoBadgeArrowDown=
'rbxassetid://111884951858150',photoBadgeArrowDownFill=
'rbxassetid://107871390721167',photoBadgeCheckmark='rbxassetid://87641114310982'
,photoBadgeCheckmarkFill='rbxassetid://135388635005016',
photoBadgeExclamationmark='rbxassetid://85889142841805',
photoBadgeExclamationmarkFill='rbxassetid://70564106880729',
photoBadgeMagnifyingglass='rbxassetid://135894823011821',
photoBadgeMagnifyingglassFill='rbxassetid://104248787134254',photoBadgePlus=
'rbxassetid://139218119153470',photoBadgePlusFill='rbxassetid://137533733758015'
,photoBadgeShieldExclamationmark='rbxassetid://93485639865954',
photoBadgeShieldExclamationmarkFill='rbxassetid://106649923225765',photoCircle=
'rbxassetid://82562621732217',photoCircleFill='rbxassetid://94004054038412',
photoFill='rbxassetid://111680668639782',photoFillOnRectangleFill=
'rbxassetid://139880248650794',photoOnRectangle='rbxassetid://74009493246173',
photoOnRectangleAngled='rbxassetid://78929588928496',photoOnRectangleAngledFill=
'rbxassetid://116615960255503',photoStack='rbxassetid://116744098593728',
photoStackFill='rbxassetid://136351647084409',photoTrianglebadgeExclamationmark=
'rbxassetid://127380965749556',photoTrianglebadgeExclamationmarkFill=
'rbxassetid://138953418202254',photoTv='rbxassetid://83951502842716',pi=
'rbxassetid://137262054651372',piCircle='rbxassetid://135373869294798',
piCircleFill='rbxassetid://120888427088049',piSquare=
'rbxassetid://87692915899283',piSquareFill='rbxassetid://134420038676982',
pianokeys='rbxassetid://100398610027172',pianokeysInverse=
'rbxassetid://79698280620441',pill='rbxassetid://126278443123311',pillCircle=
'rbxassetid://113470872116477',pillCircleFill='rbxassetid://102134863235143',
pillFill='rbxassetid://118865334913811',pills='rbxassetid://135654600545618',
pillsCircle='rbxassetid://101748290464454',pillsCircleFill=
'rbxassetid://83035294003188',pillsFill='rbxassetid://80480569153430',pin=
'rbxassetid://134143049313259',pinCircle='rbxassetid://140386216854767',
pinCircleFill='rbxassetid://125997437793325',pinFill=
'rbxassetid://105480877942834',pinSlash='rbxassetid://133152106066233',
pinSlashFill='rbxassetid://108270742635671',pinSquare=
'rbxassetid://96954048377270',pinSquareFill='rbxassetid://119024645426263',pip=
'rbxassetid://71575912606161',pipEnter='rbxassetid://128947069900942',pipExit=
'rbxassetid://100796125872672',pipFill='rbxassetid://124361284867478',pipRemove=
'rbxassetid://124134881366416',pipSwap='rbxassetid://108274050795309',
pipeAndDrop='rbxassetid://138939198499930',pipeAndDropFill=
'rbxassetid://95117828700088',placeholdertextFill='rbxassetid://96792035090493',
platter2FilledIpad='rbxassetid://128464213678375',platter2FilledIpadLandscape=
'rbxassetid://72335641279980',platter2FilledIphone=
'rbxassetid://137441342349837',platter2FilledIphoneLandscape=
'rbxassetid://85490241202606',platterBottomApplewatchCase=
'rbxassetid://89203668651377',platterFilledBottomAndArrowDownIphone=
'rbxassetid://78340878366169',platterFilledBottomApplewatchCase=
'rbxassetid://117063618828962',platterFilledBottomIphone=
'rbxassetid://89570583999672',platterFilledTopAndArrowUpIphone=
'rbxassetid://140598804745662',platterFilledTopApplewatchCase=
'rbxassetid://107739762615230',platterFilledTopIphone=
'rbxassetid://103028129852720',platterTopApplewatchCase=
'rbxassetid://127796874000587',play='rbxassetid://128461418943884',playCircle=
'rbxassetid://87454676201433',playCircleFill='rbxassetid://124194494139839',
playDesktopcomputer='rbxassetid://74705756726462',playDiamond=
'rbxassetid://131173183307365',playDiamondFill='rbxassetid://78111564875160',
playDisplay='rbxassetid://136301237559742',playFill=
'rbxassetid://74081916085124',playHouse='rbxassetid://74331836079552',
playHouseFill='rbxassetid://70509927496555',playLaptopcomputer=
'rbxassetid://112462416731590',playRectangle='rbxassetid://132181670569066',
playRectangleFill='rbxassetid://123039425293555',playRectangleOnRectangle=
'rbxassetid://91792326674527',playRectangleOnRectangleCircle=
'rbxassetid://99863288190795',playRectangleOnRectangleCircleFill=
'rbxassetid://139424729084085',playRectangleOnRectangleFill=
'rbxassetid://103121137803119',playSlash='rbxassetid://82231292976761',
playSlashFill='rbxassetid://74138387205614',playSquare=
'rbxassetid://99287717986788',playSquareFill='rbxassetid://100136882189765',
playSquareStack='rbxassetid://136134360831800',playSquareStackFill=
'rbxassetid://77854517062047',playTv='rbxassetid://127415500882482',playTvFill=
'rbxassetid://124672671078352',playpause='rbxassetid://132431310473376',
playpauseCircle='rbxassetid://72027567560609',playpauseCircleFill=
'rbxassetid://118948449910159',playpauseFill='rbxassetid://107001358735898',
playstationLogo='rbxassetid://101885651661801',plus=
'rbxassetid://104140268501180',plusApp='rbxassetid://134819696943007',
plusAppFill='rbxassetid://139827794242362',plusArrowTriangleheadClockwise=
'rbxassetid://81474708153460',plusArrowTriangleheadCounterclockwise=
'rbxassetid://90333321199077',plusBubble='rbxassetid://128128287086136',
plusBubbleFill='rbxassetid://88837705212786',plusCapsule=
'rbxassetid://82351860129396',plusCapsuleFill='rbxassetid://104771688220153',
plusCircle='rbxassetid://134903849508497',plusCircleDashed=
'rbxassetid://119427886715979',plusCircleFill='rbxassetid://126835916552120',
plusDiamond='rbxassetid://134706332882585',plusDiamondFill=
'rbxassetid://72471945998883',plusForwardslashMinus=
'rbxassetid://133024001773788',plusMagnifyingglass='rbxassetid://82229255785570'
,plusMessage='rbxassetid://139025440106833',plusMessageFill=
'rbxassetid://114656782830996',plusMinusCapsule='rbxassetid://95862408206488',
plusMinusCapsuleFill='rbxassetid://137433402562537',plusRectangle=
'rbxassetid://89928096593027',plusRectangleFill='rbxassetid://123873894242124',
plusRectangleFillOnRectangleFill='rbxassetid://84851475351254',
plusRectangleOnFolder='rbxassetid://135314696846977',plusRectangleOnFolderFill=
'rbxassetid://113831084139629',plusRectangleOnRectangle=
'rbxassetid://89954508073842',plusRectanglePortrait=
'rbxassetid://88469415424027',plusRectanglePortraitFill=
'rbxassetid://88043287673951',plusSquare='rbxassetid://132384125062898',
plusSquareDashed='rbxassetid://117358731219699',plusSquareFill=
'rbxassetid://105523739695601',plusSquareFillOnSquareFill=
'rbxassetid://102251470244460',plusSquareOnSquare='rbxassetid://112818981162820'
,plusViewfinder='rbxassetid://84219920382349',plusminus=
'rbxassetid://131496757526861',plusminusCircle='rbxassetid://75924435734916',
plusminusCircleFill='rbxassetid://139932340069454',
point3ConnectedTrianglepathDotted='rbxassetid://128994831109851',
point3FilledConnectedTrianglepathDotted='rbxassetid://87411273022672',
pointBottomleftFilledForwardToPointToprightScurvepath=
'rbxassetid://72345610995933',pointBottomleftForwardToArrowTriangleScurvepath=
'rbxassetid://121105380488921',
pointBottomleftForwardToArrowTriangleScurvepathFill=
'rbxassetid://86439950360839',
pointBottomleftForwardToArrowTriangleUturnScurvepath=
'rbxassetid://94608869275416',
pointBottomleftForwardToArrowTriangleUturnScurvepathFill=
'rbxassetid://127677527213790',
pointBottomleftForwardToPointToprightFilledScurvepath=
'rbxassetid://78467837682881',pointBottomleftForwardToPointToprightScurvepath=
'rbxassetid://105949443136809',
pointBottomleftForwardToPointToprightScurvepathFill=
'rbxassetid://90155416426752',pointForwardToPointCapsulepath=
'rbxassetid://79088115694851',pointForwardToPointCapsulepathFill=
'rbxassetid://133249842443694',pointTopleftDownToPointBottomrightCurvepath=
'rbxassetid://96359498330738',pointTopleftDownToPointBottomrightCurvepathFill=
'rbxassetid://131034155006095',pointTopleftDownToPointBottomrightFilledCurvepath
='rbxassetid://109330439433857',
pointTopleftFilledDownToPointBottomrightCurvepath='rbxassetid://119244357459528'
,pointToprightArrowTriangleBackwardToPointBottomleftFilledScurvepath=
'rbxassetid://125761722930816',
pointToprightArrowTriangleBackwardToPointBottomleftScurvepath=
'rbxassetid://101374675671818',
pointToprightArrowTriangleBackwardToPointBottomleftScurvepathFill=
'rbxassetid://95471389312772',
pointToprightFilledArrowTriangleBackwardToPointBottomleftScurvepath=
'rbxassetid://106008918139871',pointerArrow='rbxassetid://93575391711601',
pointerArrowAndSquareOnSquareDashed='rbxassetid://112982475519464',
pointerArrowClick='rbxassetid://79734702283129',pointerArrowClick2=
'rbxassetid://140079580041973',pointerArrowClickBadgeClock=
'rbxassetid://87250670163722',pointerArrowIpad='rbxassetid://98232849648450',
pointerArrowIpadAndSquareOnSquareDashed='rbxassetid://108327718703042',
pointerArrowIpadRays='rbxassetid://137810650560332',pointerArrowIpadSlash=
'rbxassetid://82866450058915',pointerArrowIpadSlashSquare=
'rbxassetid://84816174673121',pointerArrowIpadSlashSquareFill=
'rbxassetid://75197308300656',pointerArrowIpadSquare=
'rbxassetid://82086279301281',pointerArrowIpadSquareFill=
'rbxassetid://128704735814595',pointerArrowMotionlines=
'rbxassetid://82915304520768',pointerArrowMotionlinesClick=
'rbxassetid://120030717856082',pointerArrowRays='rbxassetid://122560061254398',
pointerArrowSlash='rbxassetid://115758764584094',pointerArrowSlashSquare=
'rbxassetid://94514670924898',pointerArrowSlashSquareFill=
'rbxassetid://90831995454066',pointerArrowSquare='rbxassetid://123890299712468',
pointerArrowSquareFill='rbxassetid://118743928429611',polishzlotysign=
'rbxassetid://71190576997998',
polishzlotysignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://139116746882232',polishzlotysignBankBuilding=
'rbxassetid://103554770878168',polishzlotysignBankBuildingFill=
'rbxassetid://100219158970095',polishzlotysignCircle=
'rbxassetid://133425103954783',polishzlotysignCircleFill=
'rbxassetid://85302846852426',polishzlotysignGaugeChartLefthalfRighthalf=
'rbxassetid://112861300466078',
polishzlotysignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://91379228929120',polishzlotysignRing='rbxassetid://115900998535144'
,polishzlotysignRingDashed='rbxassetid://133083266308972',polishzlotysignSquare=
'rbxassetid://116010000718053',polishzlotysignSquareFill=
'rbxassetid://126842829605701',popcorn='rbxassetid://106164924042161',
popcornCircle='rbxassetid://132014547567987',popcornCircleFill=
'rbxassetid://70873676213803',popcornFill='rbxassetid://133726972459236',power=
'rbxassetid://91472904792853',powerCircle='rbxassetid://128963948777063',
powerCircleFill='rbxassetid://88039046868989',powerDotted=
'rbxassetid://129852471510614',powercord='rbxassetid://101999056711715',
powercordFill='rbxassetid://94896872366691',powermeter=
'rbxassetid://102108395105231',poweroff='rbxassetid://98968411908379',poweron=
'rbxassetid://91641740625722',poweroutletStrip='rbxassetid://73585415920677',
poweroutletStripFill='rbxassetid://74011653309219',poweroutletTypeA=
'rbxassetid://73898196043019',poweroutletTypeAFill=
'rbxassetid://126838690734726',poweroutletTypeASquare=
'rbxassetid://100201655940379',poweroutletTypeASquareFill=
'rbxassetid://134442831578984',poweroutletTypeB='rbxassetid://120104867205082',
poweroutletTypeBFill='rbxassetid://102950626262427',poweroutletTypeBSquare=
'rbxassetid://126307738972122',poweroutletTypeBSquareFill=
'rbxassetid://114873968799324',poweroutletTypeC='rbxassetid://98862353177781',
poweroutletTypeCFill='rbxassetid://79114190266484',poweroutletTypeCSquare=
'rbxassetid://108182409003389',poweroutletTypeCSquareFill=
'rbxassetid://105015853211076',poweroutletTypeD='rbxassetid://78983099018117',
poweroutletTypeDFill='rbxassetid://131871506048623',poweroutletTypeDSquare=
'rbxassetid://78796990769278',poweroutletTypeDSquareFill=
'rbxassetid://117146065393423',poweroutletTypeE='rbxassetid://101410063610134',
poweroutletTypeEFill='rbxassetid://119773723139439',poweroutletTypeESquare=
'rbxassetid://84827194983461',poweroutletTypeESquareFill=
'rbxassetid://131818068004254',poweroutletTypeF='rbxassetid://75994498593867',
poweroutletTypeFFill='rbxassetid://108147822804278',poweroutletTypeFSquare=
'rbxassetid://114669882203509',poweroutletTypeFSquareFill=
'rbxassetid://129347734091030',poweroutletTypeG='rbxassetid://130509534014794',
poweroutletTypeGFill='rbxassetid://73751279322201',poweroutletTypeGSquare=
'rbxassetid://132654111874788',poweroutletTypeGSquareFill=
'rbxassetid://139171546199909',poweroutletTypeH='rbxassetid://121221625495909',
poweroutletTypeHFill='rbxassetid://104469280742634',poweroutletTypeHSquare=
'rbxassetid://102349433186821',poweroutletTypeHSquareFill=
'rbxassetid://87225476098525',poweroutletTypeI='rbxassetid://108432173595940',
poweroutletTypeIFill='rbxassetid://90953576052611',poweroutletTypeISquare=
'rbxassetid://85563920583968',poweroutletTypeISquareFill=
'rbxassetid://92987652110826',poweroutletTypeJ='rbxassetid://126517587735560',
poweroutletTypeJFill='rbxassetid://70372330202631',poweroutletTypeJSquare=
'rbxassetid://108455386185985',poweroutletTypeJSquareFill=
'rbxassetid://110671250365058',poweroutletTypeK='rbxassetid://90986976673417',
poweroutletTypeKFill='rbxassetid://104105946842300',poweroutletTypeKSquare=
'rbxassetid://71665389391339',poweroutletTypeKSquareFill=
'rbxassetid://100252749143408',poweroutletTypeL='rbxassetid://102082292598917',
poweroutletTypeLFill='rbxassetid://95679765765332',poweroutletTypeLSquare=
'rbxassetid://101581018956257',poweroutletTypeLSquareFill=
'rbxassetid://108089600610800',poweroutletTypeM='rbxassetid://77038437139981',
poweroutletTypeMFill='rbxassetid://102565240968296',poweroutletTypeMSquare=
'rbxassetid://137482102637754',poweroutletTypeMSquareFill=
'rbxassetid://101061604726738',poweroutletTypeN='rbxassetid://131231933791747',
poweroutletTypeNFill='rbxassetid://122521914168552',poweroutletTypeNSquare=
'rbxassetid://88193045014774',poweroutletTypeNSquareFill=
'rbxassetid://117707236613750',poweroutletTypeO='rbxassetid://71518035241990',
poweroutletTypeOFill='rbxassetid://124355652039674',poweroutletTypeOSquare=
'rbxassetid://93155191747385',poweroutletTypeOSquareFill=
'rbxassetid://75876768217678',powerplug='rbxassetid://113915951601980',
powerplugFill='rbxassetid://72971139891393',powerplugPortrait=
'rbxassetid://134444701296189',powerplugPortraitFill=
'rbxassetid://129569799503764',powersleep='rbxassetid://136441993961962',printer
='rbxassetid://92726076996600',printerDotmatrix='rbxassetid://71926967169220',
printerDotmatrixFill='rbxassetid://87728661853490',
printerDotmatrixFilledAndPaper='rbxassetid://105425549423600',
printerDotmatrixFilledAndPaperInverse='rbxassetid://74443986062725',
printerDotmatrixInverse='rbxassetid://91372868092556',printerFill=
'rbxassetid://77630139933554',printerFilledAndPaper=
'rbxassetid://97682473483014',printerFilledAndPaperInverse=
'rbxassetid://137162835775605',printerInverse='rbxassetid://96106141032867',
progressIndicator='rbxassetid://89634242697334',projective=
'rbxassetid://73010755604065',purchased='rbxassetid://89342661938554',
purchasedCircle='rbxassetid://119930548891031',purchasedCircleFill=
'rbxassetid://125749909647263',puzzlepiece='rbxassetid://127623471300702',
puzzlepieceExtension='rbxassetid://106644636346231',puzzlepieceExtensionFill=
'rbxassetid://75203726915481',puzzlepieceFill='rbxassetid://117289958016840',
pyramid='rbxassetid://112564763709051',pyramidFill='rbxassetid://92297972398994'
,qCircle='rbxassetid://134479590621303',qCircleFill=
'rbxassetid://85104022920098',qSquare='rbxassetid://126798899208180',qSquareFill
='rbxassetid://131372232713091',qrcode='rbxassetid://135429731093760',
qrcodeViewfinder='rbxassetid://70376735626959',questionmark=
'rbxassetid://135728947505125',questionmarkApp='rbxassetid://125486303792695',
questionmarkAppDashed='rbxassetid://93908191503686',questionmarkAppFill=
'rbxassetid://132390651792916',questionmarkBubble='rbxassetid://79680750661243',
questionmarkBubbleFill='rbxassetid://109480620904091',questionmarkCircle=
'rbxassetid://106563268903746',questionmarkCircleDashed=
'rbxassetid://90578083008627',questionmarkCircleFill=
'rbxassetid://84091622533619',questionmarkDiamond='rbxassetid://133039366755179'
,questionmarkDiamondFill='rbxassetid://96713509421435',questionmarkFolder=
'rbxassetid://123509903209077',questionmarkFolderFill=
'rbxassetid://109535614084769',questionmarkKeyFilled=
'rbxassetid://115154856009498',questionmarkMessage=
'rbxassetid://120735530943822',questionmarkMessageFill=
'rbxassetid://138648428289642',questionmarkSquare='rbxassetid://80342776780001',
questionmarkSquareDashed='rbxassetid://100367436719082',questionmarkSquareFill=
'rbxassetid://76990513345923',questionmarkTextPage='rbxassetid://77522269508702'
,questionmarkTextPageFill='rbxassetid://74250116725322',questionmarkVideo=
'rbxassetid://96965650538855',questionmarkVideoFill=
'rbxassetid://102872787704052',quoteBubble='rbxassetid://120785470678014',
quoteBubbleFill='rbxassetid://134548894341595',quoteClosing=
'rbxassetid://128300262716157',quoteOpening='rbxassetid://85648900225966',
quotelevel='rbxassetid://133835140047628',r1ButtonRoundedbottomHorizontal=
'rbxassetid://73494380889514',r1ButtonRoundedbottomHorizontalFill=
'rbxassetid://87732102079281',r1Circle='rbxassetid://135769834088860',
r1CircleFill='rbxassetid://130502350729327',r2ButtonAngledtopVerticalRight=
'rbxassetid://133768730605538',r2ButtonAngledtopVerticalRightFill=
'rbxassetid://88773533237354',r2ButtonRoundedtopHorizontal=
'rbxassetid://98251601517278',r2ButtonRoundedtopHorizontalFill=
'rbxassetid://106192967203243',r2Circle='rbxassetid://106631267824223',
r2CircleFill='rbxassetid://132126702469862',r3ButtonAngledbottomHorizontalRight=
'rbxassetid://137304943113639',r3ButtonAngledbottomHorizontalRightFill=
'rbxassetid://89807708696162',r4ButtonHorizontal='rbxassetid://101176118558083',
r4ButtonHorizontalFill='rbxassetid://111110114987717',
rButtonRoundedbottomHorizontal='rbxassetid://106791314037092',
rButtonRoundedbottomHorizontalFill='rbxassetid://127088507444504',rCircle=
'rbxassetid://100474397953607',rCircleFill='rbxassetid://109515288026300',
rJoystick='rbxassetid://82799323747888',rJoystickFill=
'rbxassetid://88376161432211',rJoystickPressDown='rbxassetid://130205975134988',
rJoystickPressDownFill='rbxassetid://82709360617451',rJoystickTiltDown=
'rbxassetid://125350323338775',rJoystickTiltDownFill=
'rbxassetid://122451980946728',rJoystickTiltLeft='rbxassetid://100163992653880',
rJoystickTiltLeftFill='rbxassetid://126371751528008',rJoystickTiltRight=
'rbxassetid://73216566650577',rJoystickTiltRightFill=
'rbxassetid://95154512055976',rJoystickTiltUp='rbxassetid://89422740003979',
rJoystickTiltUpFill='rbxassetid://124671519494711',rSquare=
'rbxassetid://71910946326643',rSquareFill='rbxassetid://93196796932773',
rSquareOnSquare='rbxassetid://107811193096025',rSquareOnSquareFill=
'rbxassetid://113609306294519',radio='rbxassetid://83922658753052',radioFill=
'rbxassetid://122048912816887',rainbow='rbxassetid://91453713893762',rays=
'rbxassetid://82549830905060',rbButtonRoundedbottomHorizontal=
'rbxassetid://99388233415918',rbButtonRoundedbottomHorizontalFill=
'rbxassetid://115801079364851',rbCircle='rbxassetid://138035345745553',
rbCircleFill='rbxassetid://101169325937487',receipt=
'rbxassetid://76381361847492',receiptFill='rbxassetid://89891989598979',
recordCircle='rbxassetid://108217106941598',recordCircleFill=
'rbxassetid://105698081236112',recordingtape='rbxassetid://127308726768030',
recordingtapeCircle='rbxassetid://104143200089711',recordingtapeCircleFill=
'rbxassetid://80198600120854',rectangle='rbxassetid://111414459754261',
rectangle2Swap='rbxassetid://131330634853965',rectangle3Group=
'rbxassetid://135787395657285',rectangle3GroupBubble=
'rbxassetid://116160459807032',rectangle3GroupBubbleFill=
'rbxassetid://139621481371395',rectangle3GroupDashed=
'rbxassetid://110904729324036',rectangle3GroupFill='rbxassetid://93139300368934'
,rectangleAndArrowUpRightAndArrowDownLeft='rbxassetid://93638105367907',
rectangleAndArrowUpRightAndArrowDownLeftSlash='rbxassetid://106369518263199',
rectangleAndHandPointUpLeft='rbxassetid://93683011357377',
rectangleAndHandPointUpLeftFill='rbxassetid://99025075004423',
rectangleAndHandPointUpLeftFilled='rbxassetid://76034474354386',
rectangleAndPaperclip='rbxassetid://105337301519485',
rectangleAndPencilAndEllipsis='rbxassetid://78563329334131',
rectangleAndTextMagnifyingglass='rbxassetid://127177835398113',
rectangleArrowtriangle2Inward='rbxassetid://138600951302016',
rectangleArrowtriangle2Outward='rbxassetid://136346043617802',
rectangleBadgeCheckmark='rbxassetid://72076028531820',rectangleBadgeMinus=
'rbxassetid://102402218146653',rectangleBadgePersonCrop=
'rbxassetid://70527611658257',rectangleBadgePlus='rbxassetid://106486906660779',
rectangleBadgeXmark='rbxassetid://108083111117375',rectangleBottomhalfFilled=
'rbxassetid://101103984667014',rectangleCompressVertical=
'rbxassetid://121411068286936',rectangleConnectedToLineBelow=
'rbxassetid://77423923606068',rectangleDashed='rbxassetid://138713016735236',
rectangleDashedAndPaperclip='rbxassetid://110455441885522',
rectangleDashedBadgeRecord='rbxassetid://128289633292737',
rectangleExpandDiagonal='rbxassetid://122288568531399',rectangleExpandVertical=
'rbxassetid://115245648914946',rectangleFill='rbxassetid://130588385351314',
rectangleFillBadgeCheckmark='rbxassetid://101292450070677',
rectangleFillBadgeMinus='rbxassetid://87052252529460',
rectangleFillBadgePersonCrop='rbxassetid://79555531925800',
rectangleFillBadgePlus='rbxassetid://78893754793162',rectangleFillBadgeXmark=
'rbxassetid://103536021556291',rectangleFillOnRectangleAngledFill=
'rbxassetid://91215625396644',rectangleFillOnRectangleFill=
'rbxassetid://138672467195296',rectangleFilledAndHandPointUpLeft=
'rbxassetid://129165814821096',rectangleGrid1x2='rbxassetid://90808025440357',
rectangleGrid1x2Fill='rbxassetid://73911101625046',rectangleGrid1x3=
'rbxassetid://117353042982000',rectangleGrid1x3Fill=
'rbxassetid://111448923249339',rectangleGrid2x2='rbxassetid://92889258947698',
rectangleGrid2x2Fill='rbxassetid://79589884272899',rectangleGrid3x1=
'rbxassetid://76689994608928',rectangleGrid3x1Fill=
'rbxassetid://131318740621070',rectangleGrid3x2='rbxassetid://137686973180621',
rectangleGrid3x2Fill='rbxassetid://76899837809257',rectangleGrid3x3=
'rbxassetid://111445407298994',rectangleGrid3x3Fill=
'rbxassetid://117660783834604',rectangleLandscapeRotate=
'rbxassetid://92428477483439',rectangleLandscapeRotateSlash=
'rbxassetid://82008920679309',rectangleLeadinghalfFilled=
'rbxassetid://79972096403571',rectangleLefthalfFilled=
'rbxassetid://131457794136852',rectangleOnRectangle=
'rbxassetid://125190929833969',rectangleOnRectangleAngled=
'rbxassetid://122398141357009',rectangleOnRectangleBadgeGearshape=
'rbxassetid://101252695124574',rectangleOnRectangleButtonAngledtopVerticalLeft=
'rbxassetid://124890917881545',
rectangleOnRectangleButtonAngledtopVerticalLeftFill=
'rbxassetid://122574969411469',rectangleOnRectangleCircle=
'rbxassetid://73017118558797',rectangleOnRectangleCircleFill=
'rbxassetid://106477109606197',rectangleOnRectangleDashed=
'rbxassetid://100408401861363',rectangleOnRectangleSlash=
'rbxassetid://77613589607929',rectangleOnRectangleSlashCircle=
'rbxassetid://91428157222160',rectangleOnRectangleSlashCircleFill=
'rbxassetid://140066340357199',rectangleOnRectangleSlashFill=
'rbxassetid://79680969346171',rectangleOnRectangleSquare=
'rbxassetid://71711185277015',rectangleOnRectangleSquareFill=
'rbxassetid://80274524367727',rectanglePatternCheckered=
'rbxassetid://107724057059604',rectanglePortrait='rbxassetid://125236374031015',
rectanglePortraitAndArrowForward='rbxassetid://120445430817911',
rectanglePortraitAndArrowForwardFill='rbxassetid://91657691983927',
rectanglePortraitAndArrowRight='rbxassetid://78586132872195',
rectanglePortraitAndArrowRightFill='rbxassetid://100165849200729',
rectanglePortraitArrowtriangle2Inward='rbxassetid://140269657314813',
rectanglePortraitArrowtriangle2Outward='rbxassetid://137195383116487',
rectanglePortraitBadgePlus='rbxassetid://98290274050547',
rectanglePortraitBadgePlusFill='rbxassetid://122337836606239',
rectanglePortraitBottomhalfFilled='rbxassetid://114165508753177',
rectanglePortraitFill='rbxassetid://98444426893441',
rectanglePortraitLefthalfFilled='rbxassetid://118614403436193',
rectanglePortraitOnRectanglePortrait='rbxassetid://88124841316261',
rectanglePortraitOnRectanglePortraitAngled='rbxassetid://100172422875638',
rectanglePortraitOnRectanglePortraitAngledFill='rbxassetid://131829103874258',
rectanglePortraitOnRectanglePortraitFill='rbxassetid://90599203086952',
rectanglePortraitOnRectanglePortraitSlash='rbxassetid://91665771440270',
rectanglePortraitOnRectanglePortraitSlashFill='rbxassetid://109451249077103',
rectanglePortraitRighthalfFilled='rbxassetid://100066536343390',
rectanglePortraitRotate='rbxassetid://81490858477590',
rectanglePortraitRotateSlash='rbxassetid://114808604031742',
rectanglePortraitSlash='rbxassetid://72406543095141',rectanglePortraitSlashFill=
'rbxassetid://136630733879613',rectanglePortraitSplit2x1=
'rbxassetid://87323130284872',rectanglePortraitSplit2x1Fill=
'rbxassetid://85586482314976',rectanglePortraitSplit2x1Slash=
'rbxassetid://118538465517234',rectanglePortraitSplit2x1SlashFill=
'rbxassetid://103708519113694',rectanglePortraitTophalfFilled=
'rbxassetid://108433047247888',rectangleRatio16To9=
'rbxassetid://137417821446334',rectangleRatio16To9Fill=
'rbxassetid://88975270022668',rectangleRatio3To4='rbxassetid://80074563926947',
rectangleRatio3To4Fill='rbxassetid://93688722112538',rectangleRatio4To3=
'rbxassetid://113646253383623',rectangleRatio4To3Fill=
'rbxassetid://118821190872382',rectangleRatio9To16='rbxassetid://78790871575968'
,rectangleRatio9To16Fill='rbxassetid://86914887040905',rectangleRighthalfFilled=
'rbxassetid://114536690093297',rectangleSlash='rbxassetid://93912868038495',
rectangleSlashFill='rbxassetid://125060844525663',rectangleSplit1x2=
'rbxassetid://78612147332368',rectangleSplit1x2Fill=
'rbxassetid://80761667288444',rectangleSplit2x1='rbxassetid://119372955276022',
rectangleSplit2x1Fill='rbxassetid://139921639891469',rectangleSplit2x1Slash=
'rbxassetid://104484575664164',rectangleSplit2x1SlashFill=
'rbxassetid://114315803976014',rectangleSplit2x2='rbxassetid://83495194872523',
rectangleSplit2x2Fill='rbxassetid://114784847338074',rectangleSplit3x1=
'rbxassetid://105846009511841',rectangleSplit3x1Fill=
'rbxassetid://113328142141984',rectangleSplit3x3='rbxassetid://78793707576857',
rectangleSplit3x3Fill='rbxassetid://78131312150919',rectangleStack=
'rbxassetid://105605876322301',rectangleStackBadgeMinus=
'rbxassetid://127965533772535',rectangleStackBadgePersonCrop=
'rbxassetid://129251589155547',rectangleStackBadgePersonCropFill=
'rbxassetid://79977200721454',rectangleStackBadgePlay=
'rbxassetid://103484646951126',rectangleStackBadgePlayFill=
'rbxassetid://74796728731284',rectangleStackBadgePlus=
'rbxassetid://79107569318116',rectangleStackFill='rbxassetid://135855303627369',
rectangleStackFillBadgeMinus='rbxassetid://107768535623331',
rectangleStackFillBadgePlus='rbxassetid://97606185836694',rectangleStackSlash=
'rbxassetid://104855023839912',rectangleStackSlashFill=
'rbxassetid://94713737257399',rectangleTophalfFilled=
'rbxassetid://116794907916061',rectangleTrailinghalfFilled=
'rbxassetid://116099294996666',refrigerator='rbxassetid://115470198294720',
refrigeratorFill='rbxassetid://91481483660716',['repeat']=
'rbxassetid://89312130153366',repeat1='rbxassetid://95914998265758',
repeat1Circle='rbxassetid://125048734108902',repeat1CircleFill=
'rbxassetid://103291311712495',repeatBadgeXmark='rbxassetid://136313092066440',
repeatCircle='rbxassetid://103368260994746',repeatCircleFill=
'rbxassetid://75689548597847',restart='rbxassetid://79478375893990',
restartCircle='rbxassetid://74213777320248',restartCircleFill=
'rbxassetid://93709981958260',retarderBrakesignal='rbxassetid://116812543667964'
,retarderBrakesignalAndExclamationmark='rbxassetid://128582163780524',
retarderBrakesignalSlash='rbxassetid://91774982651852',['return']=
'rbxassetid://93686227671158',returnLeft='rbxassetid://110666344633227',
returnRight='rbxassetid://125304385375369',rhombus=
'rbxassetid://130322524489833',rhombusFill='rbxassetid://137003561050469',
richtextPage='rbxassetid://101913089792940',richtextPageFill=
'rbxassetid://120660865942330',right='rbxassetid://93581059929312',rightCircle=
'rbxassetid://126673356006020',rightCircleFill='rbxassetid://107888523128870',
righttriangle='rbxassetid://98781907340244',righttriangleFill=
'rbxassetid://97751299193494',righttriangleSplitDiagonal=
'rbxassetid://140267826439171',righttriangleSplitDiagonalFill=
'rbxassetid://134486346049833',ring='rbxassetid://134891574217355',ringDashed=
'rbxassetid://79016816094406',rmButtonHorizontal='rbxassetid://112649649945799',
rmButtonHorizontalFill='rbxassetid://101555031791001',
roadLaneArrowtriangle2Inward='rbxassetid://83028812738091',
roadLaneArrowtriangle2Outward='rbxassetid://125107735583240',roadLanes=
'rbxassetid://121827579804737',roadLanesCurvedLeft=
'rbxassetid://114689746540833',roadLanesCurvedRight=
'rbxassetid://76090752203397',roboticVacuum='rbxassetid://85432635849466',
roboticVacuumAndArrowtriangleUp='rbxassetid://99360183176046',
roboticVacuumAndArrowtriangleUpFill='rbxassetid://133281032251829',
roboticVacuumAndEllipsis='rbxassetid://133526596908702',
roboticVacuumAndEllipsisFill='rbxassetid://71988250367062',roboticVacuumFill=
'rbxassetid://129505593002521',rollerShadeClosed='rbxassetid://104900591917665',
rollerShadeOpen='rbxassetid://88571033999479',romanShadeClosed=
'rbxassetid://96918750802866',romanShadeOpen='rbxassetid://90499680966105',
rosette='rbxassetid://128397706826097',rotate3d='rbxassetid://79997216476659',
rotate3dCircle='rbxassetid://134023839176423',rotate3dCircleFill=
'rbxassetid://136024177936496',rotate3dFill='rbxassetid://140259573440265',
rotateLeft='rbxassetid://104293817458403',rotateLeftFill=
'rbxassetid://108834532920454',rotateRight='rbxassetid://101811576269464',
rotateRightFill='rbxassetid://132494268034794',
rsbButtonAngledbottomHorizontalRight='rbxassetid://104443343478623',
rsbButtonAngledbottomHorizontalRightFill='rbxassetid://105946161113872',
rtButtonRoundedtopHorizontal='rbxassetid://88197217150788',
rtButtonRoundedtopHorizontalFill='rbxassetid://137834814026503',rtCircle=
'rbxassetid://130736157262549',rtCircleFill='rbxassetid://90688963975051',
rublesign='rbxassetid://79995340276002',
rublesignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://107533533773481',rublesignBankBuilding=
'rbxassetid://112367554211189',rublesignBankBuildingFill=
'rbxassetid://74276645461175',rublesignCircle='rbxassetid://138616838615062',
rublesignCircleFill='rbxassetid://117918611348729',
rublesignGaugeChartLefthalfRighthalf='rbxassetid://121997111268342',
rublesignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://124825936849158',
rublesignRing='rbxassetid://102327958941047',rublesignRingDashed=
'rbxassetid://127403954235716',rublesignSquare='rbxassetid://93428290716501',
rublesignSquareFill='rbxassetid://91069416883880',rugbyball=
'rbxassetid://107813412868735',rugbyballCircle='rbxassetid://74520104438728',
rugbyballCircleFill='rbxassetid://132264267384386',rugbyballFill=
'rbxassetid://133697647276958',ruler='rbxassetid://137893325259884',rulerFill=
'rbxassetid://95614061471677',rupeesign='rbxassetid://109206009121524',
rupeesignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://128550816369153',rupeesignBankBuilding=
'rbxassetid://105737039771832',rupeesignBankBuildingFill=
'rbxassetid://131896695518623',rupeesignCircle='rbxassetid://131458084468081',
rupeesignCircleFill='rbxassetid://76182665186164',
rupeesignGaugeChartLefthalfRighthalf='rbxassetid://125393632929329',
rupeesignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://96777336576753',
rupeesignRing='rbxassetid://125467764440847',rupeesignRingDashed=
'rbxassetid://82381851316657',rupeesignSquare='rbxassetid://111461203852733',
rupeesignSquareFill='rbxassetid://108460282949877',sCircle=
'rbxassetid://70797943728324',sCircleFill='rbxassetid://114752832731487',sSquare
='rbxassetid://97768874847459',sSquareFill='rbxassetid://124722554149961',safari
='rbxassetid://76428111544714',safariFill='rbxassetid://85821469115818',sailboat
='rbxassetid://99246226977355',sailboatCircle='rbxassetid://92706155026002',
sailboatCircleFill='rbxassetid://119525831244370',sailboatFill=
'rbxassetid://125556225154215',scale3d='rbxassetid://116105033501936',scalemass=
'rbxassetid://117780975283623',scalemassFill='rbxassetid://78891902031658',
scanner='rbxassetid://73017296718236',scannerFill='rbxassetid://86721241098861',
scissors='rbxassetid://95513121521151',scissorsBadgeEllipsis=
'rbxassetid://116175702980810',scissorsCircle='rbxassetid://100442513150877',
scissorsCircleFill='rbxassetid://133071884128532',scooter=
'rbxassetid://106371037983152',scope='rbxassetid://114797558688010',screwdriver=
'rbxassetid://128552531928875',screwdriverFill='rbxassetid://111697095003710',
scribble='rbxassetid://73635938346366',scribbleVariable=
'rbxassetid://95857213844977',scroll='rbxassetid://100024126240372',scrollFill=
'rbxassetid://86513801307528',sdcard='rbxassetid://91999340306150',sdcardFill=
'rbxassetid://86816824579142',seal='rbxassetid://112968110857352',sealFill=
'rbxassetid://111595007269811',selectionPinInOut='rbxassetid://81834666710003',
sensor='rbxassetid://79957599284236',sensorFill='rbxassetid://100348356748640',
sensorRadiowavesLeftAndRight='rbxassetid://137434009722117',
sensorRadiowavesLeftAndRightFill='rbxassetid://106680582974812',
sensorTagRadiowavesForward='rbxassetid://106463263648802',
sensorTagRadiowavesForwardFill='rbxassetid://99279214867737',serverRack=
'rbxassetid://83940756291905',serviceDog='rbxassetid://139343512896351',
serviceDogFill='rbxassetid://81589591908524',shadow=
'rbxassetid://115420710489840',sharedwithyou='rbxassetid://98071581699327',
sharedwithyouCircle='rbxassetid://76114224177131',sharedwithyouCircleFill=
'rbxassetid://81557681633469',sharedwithyouSlash='rbxassetid://86347172828564',
shareplay='rbxassetid://102891055337180',shareplaySlash=
'rbxassetid://116971365339623',shazamLogo='rbxassetid://92839379188348',
shazamLogoFill='rbxassetid://103713552773129',shekelsign=
'rbxassetid://129156275836476',
shekelsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://88176105509519',shekelsignBankBuilding=
'rbxassetid://83980545208734',shekelsignBankBuildingFill=
'rbxassetid://75899516654392',shekelsignCircle='rbxassetid://118065137678244',
shekelsignCircleFill='rbxassetid://110241676766230',
shekelsignGaugeChartLefthalfRighthalf='rbxassetid://138077316594571',
shekelsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://106188539296043',
shekelsignRing='rbxassetid://96814236600555',shekelsignRingDashed=
'rbxassetid://77084151086437',shekelsignSquare='rbxassetid://98662725701428',
shekelsignSquareFill='rbxassetid://138075137740999',shield=
'rbxassetid://140526681233529',shieldFill='rbxassetid://124747829928806',
shieldLefthalfFilled='rbxassetid://117577138863097',
shieldLefthalfFilledBadgeCheckmark='rbxassetid://87003650517234',
shieldLefthalfFilledSlash='rbxassetid://135125468470118',
shieldLefthalfFilledTrianglebadgeExclamationmark='rbxassetid://72778955082789',
shieldPatternCheckered='rbxassetid://130283063693178',shieldRighthalfFilled=
'rbxassetid://121059176356852',shieldSlash='rbxassetid://110010335990680',
shieldSlashFill='rbxassetid://90727030243014',shift=
'rbxassetid://139605024762137',shiftFill='rbxassetid://103534834680447',
shippingbox='rbxassetid://135179382969903',shippingboxAndArrowBackward=
'rbxassetid://121360973310421',shippingboxAndArrowBackwardFill=
'rbxassetid://93727205953096',shippingboxCircle='rbxassetid://96682399196466',
shippingboxCircleFill='rbxassetid://132322216128885',shippingboxFill=
'rbxassetid://90240043893822',shoe='rbxassetid://70988953478341',shoe2=
'rbxassetid://103120492305654',shoe2Fill='rbxassetid://140691495305840',
shoeArrowTriangleheadUpAndDown='rbxassetid://127847462147663',
shoeArrowTriangleheadUpAndDownFill='rbxassetid://110267382636824',
shoeArrowTriangleheadUpRight='rbxassetid://84029339972591',
shoeArrowTriangleheadUpRightCircle='rbxassetid://130133214377534',
shoeArrowTriangleheadUpRightCircleFill='rbxassetid://83387140921529',
shoeArrowTriangleheadUpRightFill='rbxassetid://114661642297786',shoeCircle=
'rbxassetid://139385430751379',shoeCircleFill='rbxassetid://96913294618430',
shoeFill='rbxassetid://70447777963358',shoeprintsFill=
'rbxassetid://83128922492077',shower='rbxassetid://136336766208797',showerFill=
'rbxassetid://128574689734106',showerHandheld='rbxassetid://123682140353662',
showerHandheldFill='rbxassetid://105769933058151',showerSidejet=
'rbxassetid://79554964698952',showerSidejetFill='rbxassetid://127129458580502',
shuffle='rbxassetid://75346543378152',shuffleCircle=
'rbxassetid://116411175114787',shuffleCircleFill='rbxassetid://102012828566108',
sidebarLeading='rbxassetid://83956276332602',sidebarLeft=
'rbxassetid://71643705896266',sidebarRight='rbxassetid://114611117811607',
sidebarSquaresLeading='rbxassetid://126932130215814',sidebarSquaresLeft=
'rbxassetid://102411145840494',sidebarSquaresRight='rbxassetid://81074371398892'
,sidebarSquaresTrailing='rbxassetid://132148034874544',sidebarTrailing=
'rbxassetid://95429731716641',signature='rbxassetid://110131488283477',
signpostAndArrowtriangleUp='rbxassetid://97583682433894',
signpostAndArrowtriangleUpCircle='rbxassetid://108385973369296',
signpostAndArrowtriangleUpCircleFill='rbxassetid://107743240413148',
signpostAndArrowtriangleUpFill='rbxassetid://138825662345098',signpostLeft=
'rbxassetid://95412606669027',signpostLeftCircle='rbxassetid://115318963051502',
signpostLeftCircleFill='rbxassetid://91336762767303',signpostLeftFill=
'rbxassetid://75163090487583',signpostRight='rbxassetid://101774272755085',
signpostRightAndLeft='rbxassetid://113062367181841',signpostRightAndLeftCircle=
'rbxassetid://81815780687045',signpostRightAndLeftCircleFill=
'rbxassetid://89749102061487',signpostRightAndLeftFill=
'rbxassetid://72053416932561',signpostRightCircle='rbxassetid://128089922540621'
,signpostRightCircleFill='rbxassetid://81406314178274',signpostRightFill=
'rbxassetid://136784047816620',simcard='rbxassetid://102108051244372',simcard2=
'rbxassetid://138566470635829',simcard2Fill='rbxassetid://103016244826117',
simcardFill='rbxassetid://126517779550731',singaporedollarsign=
'rbxassetid://134448746581237',
singaporedollarsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://76846054562178',singaporedollarsignBankBuilding=
'rbxassetid://72931926178709',singaporedollarsignBankBuildingFill=
'rbxassetid://107456617296294',singaporedollarsignCircle=
'rbxassetid://112775243884034',singaporedollarsignCircleFill=
'rbxassetid://88834107058222',singaporedollarsignGaugeChartLefthalfRighthalf=
'rbxassetid://105134570441636',
singaporedollarsignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://119200364920688',singaporedollarsignRing=
'rbxassetid://120499586212229',singaporedollarsignRingDashed=
'rbxassetid://134361953001558',singaporedollarsignSquare=
'rbxassetid://137523797948046',singaporedollarsignSquareFill=
'rbxassetid://117230157948981',sink='rbxassetid://97332507094909',sinkFill=
'rbxassetid://119601343432289',siri='rbxassetid://78943861759065',skateboard=
'rbxassetid://96609489094400',skateboardFill='rbxassetid://125654589784907',skew
='rbxassetid://103633680634238',skis='rbxassetid://114314985218926',skisFill=
'rbxassetid://128694681935126',slashCircle='rbxassetid://91525074193799',
slashCircleFill='rbxassetid://113050340993198',sleep=
'rbxassetid://112657466251253',sleepCircle='rbxassetid://135948050894103',
sleepCircleFill='rbxassetid://110524174707079',
sliderHorizontal2ArrowTriangleheadCounterclockwise='rbxassetid://86778232183781'
,sliderHorizontal2RectangleAndArrowTrianglehead2ClockwiseRotate90=
'rbxassetid://86330070759013',sliderHorizontal2Square=
'rbxassetid://90937235818005',sliderHorizontal2SquareBadgeArrowDown=
'rbxassetid://110530322362077',sliderHorizontal2SquareOnSquare=
'rbxassetid://102145308436014',sliderHorizontal3='rbxassetid://99890330649872',
sliderHorizontalBelowCircleLefthalfFilled='rbxassetid://104764044327011',
sliderHorizontalBelowCircleLefthalfFilledInverse='rbxassetid://138896643212114',
sliderHorizontalBelowCircleRighthalfFilled='rbxassetid://115251627584565',
sliderHorizontalBelowCircleRighthalfFilledInverse='rbxassetid://83816451711529',
sliderHorizontalBelowRectangle='rbxassetid://119305687087086',
sliderHorizontalBelowSquareAndSquareFilled='rbxassetid://120455732453266',
sliderHorizontalBelowSquareFilledAndSquare='rbxassetid://129990694640165',
sliderHorizontalBelowSunMax='rbxassetid://105031304876055',sliderVertical3=
'rbxassetid://102020200694699',slowmo='rbxassetid://114262094447775',
smallcircleCircle='rbxassetid://120436203127403',smallcircleCircleFill=
'rbxassetid://102648410553101',smallcircleFilledCircle=
'rbxassetid://85691773230977',smallcircleFilledCircleFill=
'rbxassetid://111196996738405',smartphone='rbxassetid://93476228253050',smoke=
'rbxassetid://104871896251732',smokeCircle='rbxassetid://108737671657911',
smokeCircleFill='rbxassetid://129640575733880',smokeFill=
'rbxassetid://127452968050922',snowboard='rbxassetid://113780393933543',
snowboardFill='rbxassetid://108936879402105',snowflake=
'rbxassetid://125255611954484',snowflakeCircle='rbxassetid://113404635828886',
snowflakeCircleFill='rbxassetid://137890067722669',snowflakeRoadLane=
'rbxassetid://100143850770605',snowflakeRoadLaneDashed=
'rbxassetid://104817339834131',snowflakeSlash='rbxassetid://99651415315718',
soccerball='rbxassetid://125849468854224',soccerballCircle=
'rbxassetid://130263604545849',soccerballCircleFill=
'rbxassetid://119282850587959',soccerballCircleFillInverse=
'rbxassetid://112192357428946',soccerballCircleInverse=
'rbxassetid://74573317455595',soccerballInverse='rbxassetid://98858584469175',
sofa='rbxassetid://120600522494597',sofaFill='rbxassetid://114779564549333',sos=
'rbxassetid://84474540582122',sosCircle='rbxassetid://79680196200371',
sosCircleFill='rbxassetid://133944108925355',space='rbxassetid://88505792015757'
,sparkle='rbxassetid://105137821962797',sparkleMagnifyingglass=
'rbxassetid://103946109249635',sparkleTextClipboard=
'rbxassetid://110580783587021',sparkleTextClipboardFill=
'rbxassetid://106843999265230',sparkles='rbxassetid://105054419961789',sparkles2
='rbxassetid://101006703552517',sparklesRectangleStack=
'rbxassetid://86520437063065',sparklesRectangleStackFill=
'rbxassetid://130420912432593',sparklesSquareFilledOnSquare=
'rbxassetid://70460348597208',sparklesTv='rbxassetid://134275091244818',
sparklesTvFill='rbxassetid://82800085062329',spatialCapture=
'rbxassetid://127355928805920',spatialCaptureFill='rbxassetid://115385600235982'
,spatialCaptureOnHexagon='rbxassetid://105807064072492',
spatialCaptureOnHexagonFill='rbxassetid://98568195175603',spatialCaptureSlash=
'rbxassetid://83328140433280',spatialCaptureSlashFill=
'rbxassetid://77330821689586',speaker='rbxassetid://90795640862724',
speakerBadgeExclamationmark='rbxassetid://79417919763353',
speakerBadgeExclamationmarkFill='rbxassetid://101773223278790',speakerCircle=
'rbxassetid://79358452707141',speakerCircleFill='rbxassetid://80055212262386',
speakerFill='rbxassetid://97746085265210',speakerMinus=
'rbxassetid://87806168928121',speakerMinusFill='rbxassetid://82240551019132',
speakerPlus='rbxassetid://84033072891945',speakerPlusFill=
'rbxassetid://102724697463555',speakerSlash='rbxassetid://111943193498500',
speakerSlashCircle='rbxassetid://90600051448985',speakerSlashCircleFill=
'rbxassetid://130708719346289',speakerSlashFill='rbxassetid://110093423263690',
speakerSquare='rbxassetid://71945065645813',speakerSquareFill=
'rbxassetid://129912752069784',speakerTrianglebadgeExclamationmark=
'rbxassetid://98367643133076',speakerTrianglebadgeExclamationmarkFill=
'rbxassetid://139026865817721',speakerWave1='rbxassetid://76621154546819',
speakerWave1ArrowtrianglesUpRightDownLeft='rbxassetid://73074603671410',
speakerWave1Fill='rbxassetid://117491707887443',speakerWave2=
'rbxassetid://93687295077826',speakerWave2Bubble='rbxassetid://118881789849031',
speakerWave2BubbleFill='rbxassetid://106305154684153',speakerWave2Circle=
'rbxassetid://84575739119409',speakerWave2CircleFill=
'rbxassetid://85071023796255',speakerWave2Fill='rbxassetid://103180330131669',
speakerWave3='rbxassetid://79826224325031',speakerWave3Fill=
'rbxassetid://114293469707588',speakerZzz='rbxassetid://118545156089427',
speakerZzzFill='rbxassetid://86985126077561',spigot=
'rbxassetid://106805522407528',spigotFill='rbxassetid://99217504785092',
spoonServing='rbxassetid://138932624792540',sportscourt=
'rbxassetid://82512117398294',sportscourtCircle='rbxassetid://99070240252051',
sportscourtCircleFill='rbxassetid://74270844136365',sportscourtFill=
'rbxassetid://135388374191975',sprinkler='rbxassetid://129645897217574',
sprinklerAndDroplets='rbxassetid://102114964726064',sprinklerAndDropletsFill=
'rbxassetid://97896768650190',sprinklerFill='rbxassetid://73036232822178',square
='rbxassetid://85143053995767',square2Layers3d='rbxassetid://70552485257532',
square2Layers3dBottomFilled='rbxassetid://125761993163696',square2Layers3dFill=
'rbxassetid://124738920278180',square2Layers3dTopFilled=
'rbxassetid://136259102006947',square3Layers3d='rbxassetid://115210073684938',
square3Layers3dBottomFilled='rbxassetid://129811646195969',
square3Layers3dDownBackward='rbxassetid://72949225714356',
square3Layers3dDownForward='rbxassetid://135710811999897',
square3Layers3dDownLeft='rbxassetid://106845150399737',
square3Layers3dDownLeftSlash='rbxassetid://127231103569358',
square3Layers3dDownRight='rbxassetid://71437542849844',
square3Layers3dDownRightSlash='rbxassetid://76582107849526',
square3Layers3dMiddleFilled='rbxassetid://123108929754648',square3Layers3dSlash=
'rbxassetid://101399839734504',square3Layers3dTopFilled=
'rbxassetid://106415948057755',squareAndArrowDown='rbxassetid://128746411462402'
,squareAndArrowDownBadgeCheckmark='rbxassetid://130768898099008',
squareAndArrowDownBadgeCheckmarkFill='rbxassetid://86242132249455',
squareAndArrowDownBadgeClock='rbxassetid://121820372914588',
squareAndArrowDownBadgeClockFill='rbxassetid://72247714968114',
squareAndArrowDownBadgeXmark='rbxassetid://105432945173625',
squareAndArrowDownBadgeXmarkFill='rbxassetid://75948895538876',
squareAndArrowDownFill='rbxassetid://120212545963289',squareAndArrowDownOnSquare
='rbxassetid://119110082141361',squareAndArrowDownOnSquareFill=
'rbxassetid://110434202328008',squareAndArrowUp='rbxassetid://113313553903045',
squareAndArrowUpBadgeCheckmark='rbxassetid://89060462360490',
squareAndArrowUpBadgeCheckmarkFill='rbxassetid://87427475669342',
squareAndArrowUpBadgeClock='rbxassetid://132739866805226',
squareAndArrowUpBadgeClockFill='rbxassetid://91052085672226',
squareAndArrowUpCircle='rbxassetid://103969691300137',squareAndArrowUpCircleFill
='rbxassetid://76698142129351',squareAndArrowUpFill=
'rbxassetid://90828927970385',squareAndArrowUpOnSquare=
'rbxassetid://135435626478341',squareAndArrowUpOnSquareFill=
'rbxassetid://95336766785015',squareAndArrowUpTrianglebadgeExclamationmark=
'rbxassetid://109390466933337',squareAndArrowUpTrianglebadgeExclamationmarkFill=
'rbxassetid://139318666375380',squareAndAtRectangle=
'rbxassetid://119683592096853',squareAndAtRectangleFill=
'rbxassetid://104255435377085',squareAndLineVerticalAndSquare=
'rbxassetid://79431983646769',squareAndLineVerticalAndSquareFilled=
'rbxassetid://112081990601652',squareAndPencil='rbxassetid://76634037563084',
squareAndPencilCircle='rbxassetid://139505288481527',squareAndPencilCircleFill=
'rbxassetid://82248531304358',squareArrowtriangle4Outward=
'rbxassetid://77298189568568',squareBadgePlus='rbxassetid://98210383623230',
squareBadgePlusFill='rbxassetid://103356838075193',squareBottomhalfFilled=
'rbxassetid://96604220147547',squareCircle='rbxassetid://139790165393025',
squareCircleFill='rbxassetid://103079992812239',squareDashed=
'rbxassetid://119064141453799',squareDotted='rbxassetid://117586347327317',
squareFill='rbxassetid://89837132511355',squareFillAndLineVerticalAndSquareFill=
'rbxassetid://75144623179474',squareFillOnCircleFill=
'rbxassetid://115476599238356',squareFillOnSquareFill=
'rbxassetid://89417266423147',squareFillTextGrid1x2=
'rbxassetid://84290036715096',squareFilledAndLineVerticalAndSquare=
'rbxassetid://80885900269249',squareFilledOnSquare=
'rbxassetid://124428264818823',squareGrid2x2='rbxassetid://80913360698670',
squareGrid2x2Fill='rbxassetid://71760249952566',squareGrid3x1BelowLineGrid1x2=
'rbxassetid://81912688426584',squareGrid3x1BelowLineGrid1x2Fill=
'rbxassetid://119653016618376',squareGrid3x1FolderBadgePlus=
'rbxassetid://74842079191055',squareGrid3x1FolderFillBadgePlus=
'rbxassetid://106031789047089',squareGrid3x2='rbxassetid://97502417328341',
squareGrid3x2Fill='rbxassetid://95159780378426',squareGrid3x3=
'rbxassetid://87652573110584',squareGrid3x3BottomleftFilled=
'rbxassetid://130862064876074',squareGrid3x3BottommiddleFilled=
'rbxassetid://116581882016989',squareGrid3x3BottomrightFilled=
'rbxassetid://137824218288354',squareGrid3x3Fill='rbxassetid://110325541887257',
squareGrid3x3MiddleFilled='rbxassetid://99768140128642',
squareGrid3x3MiddleleftFilled='rbxassetid://131771722713881',
squareGrid3x3MiddlerightFilled='rbxassetid://128882572939518',
squareGrid3x3Square='rbxassetid://107787495576279',
squareGrid3x3SquareBadgeEllipsis='rbxassetid://96252060080671',
squareGrid3x3TopleftFilled='rbxassetid://111902279638097',
squareGrid3x3TopmiddleFilled='rbxassetid://100251003844223',
squareGrid3x3ToprightFilled='rbxassetid://77522429310417',squareGrid4x3Fill=
'rbxassetid://89822928143221',squareLefthalfFilled=
'rbxassetid://123727097279981',squareOnCircle='rbxassetid://87453442013043',
squareOnSquare='rbxassetid://112515729133848',squareOnSquareBadgePersonCrop=
'rbxassetid://81628150473526',squareOnSquareBadgePersonCropFill=
'rbxassetid://70798178911051',squareOnSquareDashed='rbxassetid://85708721830387'
,squareOnSquareIntersectionDashed='rbxassetid://123685576963261',
squareOnSquareSquareshapeControlhandles='rbxassetid://132317434182442',
squareResize='rbxassetid://103137389011632',squareResizeDown=
'rbxassetid://131580684994902',squareResizeUp='rbxassetid://96408134862696',
squareRighthalfFilled='rbxassetid://83773994148405',squareSlash=
'rbxassetid://121563515816803',squareSlashFill='rbxassetid://81337393217510',
squareSplit1x2='rbxassetid://102758119479068',squareSplit1x2Fill=
'rbxassetid://126457976074520',squareSplit2x1='rbxassetid://85495348203340',
squareSplit2x1Fill='rbxassetid://99719747307469',squareSplit2x2=
'rbxassetid://103048676495294',squareSplit2x2Fill='rbxassetid://91776792312073',
squareSplitBottomrightquarter='rbxassetid://78077560669550',
squareSplitBottomrightquarterFill='rbxassetid://103695791513241',
squareSplitDiagonal='rbxassetid://107073898601138',squareSplitDiagonal2x2=
'rbxassetid://87887098327053',squareSplitDiagonal2x2Fill=
'rbxassetid://115495101484518',squareSplitDiagonalFill=
'rbxassetid://123204849485732',squareStack='rbxassetid://112149539989443',
squareStack3dDownForward='rbxassetid://123371888588139',
squareStack3dDownForwardFill='rbxassetid://92818923054422',
squareStack3dDownRight='rbxassetid://126509104158127',squareStack3dDownRightFill
='rbxassetid://101922856349607',squareStack3dForwardDottedline=
'rbxassetid://77002442081347',squareStack3dForwardDottedlineFill=
'rbxassetid://136457639194171',squareStack3dUp='rbxassetid://117987207006038',
squareStack3dUpBadgeAutomatic='rbxassetid://135930328075691',
squareStack3dUpBadgeAutomaticFill='rbxassetid://113654146090095',
squareStack3dUpFill='rbxassetid://91370889810852',squareStack3dUpSlash=
'rbxassetid://115258770745856',squareStack3dUpSlashFill=
'rbxassetid://126902421297621',squareStack3dUpTrianglebadgeExclamationmark=
'rbxassetid://75728258130488',squareStack3dUpTrianglebadgeExclamationmarkFill=
'rbxassetid://94716024705571',squareStackFill='rbxassetid://122760509196742',
squareTextSquare='rbxassetid://130921034757762',squareTextSquareFill=
'rbxassetid://84685736764504',squareTophalfFilled='rbxassetid://104415472214123'
,squareroot='rbxassetid://122874159989395',squaresBelowRectangle=
'rbxassetid://102583434437834',squaresLeadingRectangle=
'rbxassetid://84658886266010',squaresLeadingRectangleFill=
'rbxassetid://77216416187205',squareshape='rbxassetid://108626932621362',
squareshapeControlhandlesOnSquareshapeControlhandles=
'rbxassetid://136325582686841',squareshapeDottedSquareshape=
'rbxassetid://92681355705894',squareshapeFill='rbxassetid://140350880119310',
squareshapeSplit2x2='rbxassetid://107381495625763',
squareshapeSplit2x2DottedInside='rbxassetid://137947623907113',
squareshapeSplit2x2DottedInsideAndOutside='rbxassetid://120474492600718',
squareshapeSplit2x2DottedOutside='rbxassetid://123897485566440',
squareshapeSplit3x3='rbxassetid://133278932515243',squareshapeSquareshapeDotted=
'rbxassetid://108989106976024',stairs='rbxassetid://70523530478444',star=
'rbxassetid://97208341574073',starBubble='rbxassetid://137118519724278',
starBubbleFill='rbxassetid://95375870183734',starCircle=
'rbxassetid://74436398890361',starCircleFill='rbxassetid://117124119701813',
starFill='rbxassetid://112524490690853',starHexagon=
'rbxassetid://94649353014438',starHexagonFill='rbxassetid://97287319969332',
starLeadinghalfFilled='rbxassetid://76971128862533',starSlash=
'rbxassetid://70849976350941',starSlashFill='rbxassetid://134310116619944',
starSquare='rbxassetid://91348274885593',starSquareFill=
'rbxassetid://100772579100177',starSquareOnSquare='rbxassetid://70718742076403',
starSquareOnSquareFill='rbxassetid://119810588488900',staroflife=
'rbxassetid://114798584744904',staroflifeCircle='rbxassetid://76564007357028',
staroflifeCircleFill='rbxassetid://111984257276107',staroflifeFill=
'rbxassetid://71371042350725',staroflifeShield='rbxassetid://103781038038265',
staroflifeShieldFill='rbxassetid://120953658795745',steeringwheel=
'rbxassetid://98041075523701',steeringwheelAndHands=
'rbxassetid://87829430123778',steeringwheelAndHeatWaves=
'rbxassetid://118659521340187',steeringwheelAndKey=
'rbxassetid://114365366703405',steeringwheelAndLiquidWave=
'rbxassetid://132962907493387',
steeringwheelArrowTriangleheadCounterclockwiseAndClockwise=
'rbxassetid://114349664849833',steeringwheelArrowtriangleLeft=
'rbxassetid://133696680147741',steeringwheelArrowtriangleRight=
'rbxassetid://119016823583713',steeringwheelBadgeExclamationmark=
'rbxassetid://138427420451609',steeringwheelBadgeLock=
'rbxassetid://126529278950442',steeringwheelCircle=
'rbxassetid://124168726734871',steeringwheelCircleFill=
'rbxassetid://107253978218132',steeringwheelExclamationmark=
'rbxassetid://136154239115325',steeringwheelRoadLane=
'rbxassetid://89255699549572',steeringwheelRoadLaneDashed=
'rbxassetid://132868959701894',steeringwheelSlash='rbxassetid://91448583789774',
sterlingsign='rbxassetid://126562532499500',
sterlingsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://136352701272619',sterlingsignBankBuilding=
'rbxassetid://129465260320590',sterlingsignBankBuildingFill=
'rbxassetid://89415549253328',sterlingsignCircle='rbxassetid://128090877950956',
sterlingsignCircleFill='rbxassetid://96577595539313',
sterlingsignGaugeChartLefthalfRighthalf='rbxassetid://74815105245464',
sterlingsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://90408777058185',
sterlingsignRing='rbxassetid://109328496049218',sterlingsignRingDashed=
'rbxassetid://126044081571443',sterlingsignSquare='rbxassetid://123909441953591'
,sterlingsignSquareFill='rbxassetid://97561315012730',stethoscope=
'rbxassetid://108847491450617',stethoscopeCircle='rbxassetid://112416231109184',
stethoscopeCircleFill='rbxassetid://133482297875986',stop=
'rbxassetid://132098862323602',stopCircle='rbxassetid://128321680998373',
stopCircleFill='rbxassetid://118901612585535',stopFill=
'rbxassetid://74225402177219',stopwatch='rbxassetid://98740179195407',
stopwatchFill='rbxassetid://107592823329819',storefront=
'rbxassetid://135841176957124',storefrontCircle='rbxassetid://74529585747704',
storefrontCircleFill='rbxassetid://82510167755966',storefrontFill=
'rbxassetid://129684177323455',stove='rbxassetid://117736045694658',stoveFill=
'rbxassetid://106652953751053',strikethrough='rbxassetid://119273162148826',
strikethroughDouble='rbxassetid://102575177328559',strokeLineDiagonal=
'rbxassetid://77560205356047',strokeLineDiagonalSlash=
'rbxassetid://140086402893267',stroller='rbxassetid://133514081157613',
strollerFill='rbxassetid://132666279884023',studentdesk=
'rbxassetid://81200616483866',suitClub='rbxassetid://115355448500067',
suitClubFill='rbxassetid://106423658416550',suitDiamond=
'rbxassetid://102316895916267',suitDiamondFill='rbxassetid://136303405954770',
suitHeart='rbxassetid://102805929221332',suitHeartFill=
'rbxassetid://122686479848611',suitSpade='rbxassetid://85616337997149',
suitSpadeFill='rbxassetid://88147851638314',suitcase=
'rbxassetid://87609836872373',suitcaseCart='rbxassetid://125186079383688',
suitcaseCartFill='rbxassetid://80277015090553',suitcaseCircle=
'rbxassetid://119900683591433',suitcaseCircleFill='rbxassetid://135347326350559'
,suitcaseFill='rbxassetid://136958536826666',suitcaseRolling=
'rbxassetid://75357230115730',suitcaseRollingAndFilm=
'rbxassetid://122394094533244',suitcaseRollingAndFilmCircle=
'rbxassetid://97301794440482',suitcaseRollingAndFilmCircleFill=
'rbxassetid://99640827791378',suitcaseRollingAndFilmFill=
'rbxassetid://93468357058716',suitcaseRollingAndSuitcase=
'rbxassetid://121055774157302',suitcaseRollingAndSuitcaseCircle=
'rbxassetid://121378802217257',suitcaseRollingAndSuitcaseCircleFill=
'rbxassetid://92889078287104',suitcaseRollingAndSuitcaseFill=
'rbxassetid://104192984670894',suitcaseRollingCircle=
'rbxassetid://129768986866053',suitcaseRollingCircleFill=
'rbxassetid://73955642672556',suitcaseRollingFill='rbxassetid://74877686253838',
sum='rbxassetid://121256596249395',sunDust='rbxassetid://103445379416568',
sunDustCircle='rbxassetid://121983227528375',sunDustCircleFill=
'rbxassetid://89732002886704',sunDustFill='rbxassetid://80641416859494',sunHaze=
'rbxassetid://135475190059548',sunHazeCircle='rbxassetid://85520066827867',
sunHazeCircleFill='rbxassetid://114146986427142',sunHazeFill=
'rbxassetid://119817446395343',sunHorizon='rbxassetid://84120651165632',
sunHorizonCircle='rbxassetid://94640978320968',sunHorizonCircleFill=
'rbxassetid://73242682466487',sunHorizonFill='rbxassetid://100176954875747',
sunLefthalfFilled='rbxassetid://99948010377860',sunMax=
'rbxassetid://136191950602850',sunMaxCircle='rbxassetid://84176626743969',
sunMaxCircleFill='rbxassetid://135937714886387',sunMaxFill=
'rbxassetid://129021699626953',sunMaxTrianglebadgeExclamationmark=
'rbxassetid://135358000268431',sunMaxTrianglebadgeExclamationmarkFill=
'rbxassetid://99021447947882',sunMin='rbxassetid://103991364155847',sunMinFill=
'rbxassetid://106524277066060',sunRain='rbxassetid://117346545789896',
sunRainCircle='rbxassetid://116714247993788',sunRainCircleFill=
'rbxassetid://72120512860534',sunRainFill='rbxassetid://119708329959073',
sunRighthalfFilled='rbxassetid://130384758326055',sunSnow=
'rbxassetid://140218841895149',sunSnowCircle='rbxassetid://85080431656012',
sunSnowCircleFill='rbxassetid://139753867511118',sunSnowFill=
'rbxassetid://79961567888677',sunglasses='rbxassetid://139478408108550',
sunglassesFill='rbxassetid://83172262555556',sunrise=
'rbxassetid://84294788506747',sunriseCircle='rbxassetid://85082595836648',
sunriseCircleFill='rbxassetid://134981377023180',sunriseFill=
'rbxassetid://88724560564927',sunset='rbxassetid://84258742471324',sunsetCircle=
'rbxassetid://74528127670070',sunsetCircleFill='rbxassetid://111819138531245',
sunsetFill='rbxassetid://72039785697529',surfboard=
'rbxassetid://140238778372965',surfboardFill='rbxassetid://103281693034156',
suspensionShock='rbxassetid://82815963336131',suvSide=
'rbxassetid://113818306073178',suvSideAirCirculate=
'rbxassetid://101209179997042',suvSideAirCirculateFill=
'rbxassetid://95604708193928',suvSideAirFresh='rbxassetid://119537195789706',
suvSideAirFreshFill='rbxassetid://128602482925386',suvSideAndExclamationmark=
'rbxassetid://70935742116781',suvSideAndExclamationmarkFill=
'rbxassetid://134726230610351',suvSideArrowLeftAndRight=
'rbxassetid://139550769625928',suvSideArrowLeftAndRightFill=
'rbxassetid://135409490346330',suvSideArrowtriangleDown=
'rbxassetid://71707878766537',suvSideArrowtriangleDownFill=
'rbxassetid://88284067475836',suvSideArrowtriangleUp=
'rbxassetid://109244877917656',suvSideArrowtriangleUpArrowtriangleDown=
'rbxassetid://118056351833978',suvSideArrowtriangleUpArrowtriangleDownFill=
'rbxassetid://125999031098593',suvSideArrowtriangleUpFill=
'rbxassetid://90910642207609',suvSideFill='rbxassetid://82704988425727',
suvSideFrontOpen='rbxassetid://72250378572119',suvSideFrontOpenCrop=
'rbxassetid://120980559843625',suvSideFrontOpenCropFill=
'rbxassetid://125542014790146',suvSideFrontOpenFill=
'rbxassetid://132460379724922',suvSideHillDescentControl=
'rbxassetid://126063797863800',suvSideHillDescentControlFill=
'rbxassetid://77045237271977',suvSideHillDown='rbxassetid://134786730212068',
suvSideHillDownAndGaugeOpenWithLinesNeedle25percentAndArrowtriangle=
'rbxassetid://83945172818973',
suvSideHillDownAndGaugeOpenWithLinesNeedle25percentAndArrowtriangleFill=
'rbxassetid://114026491508514',suvSideHillDownFill='rbxassetid://72730443837671'
,suvSideHillUp='rbxassetid://129652117525451',suvSideHillUpFill=
'rbxassetid://134588895330123',suvSideLock='rbxassetid://135078968309336',
suvSideLockFill='rbxassetid://123003915936036',suvSideLockOpen=
'rbxassetid://115230659352541',suvSideLockOpenFill=
'rbxassetid://129135935833817',suvSideRearOpen='rbxassetid://139226380860692',
suvSideRearOpenCrop='rbxassetid://88433095209406',suvSideRearOpenCropFill=
'rbxassetid://71941757962126',suvSideRearOpenFill='rbxassetid://121255200757213'
,suvSideRoofCargoCarrier='rbxassetid://90680771893599',
suvSideRoofCargoCarrierFill='rbxassetid://133507228256677',
suvSideRoofCargoCarrierSlash='rbxassetid://73339283777162',
suvSideRoofCargoCarrierSlashFill='rbxassetid://72759140430639',swatchpalette=
'rbxassetid://125516289768416',swatchpaletteFill='rbxassetid://111232642263672',
swedishkronasign='rbxassetid://115059054061881',
swedishkronasignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://107015130866436',swedishkronasignBankBuilding=
'rbxassetid://94661078864168',swedishkronasignBankBuildingFill=
'rbxassetid://122674758234286',swedishkronasignCircle=
'rbxassetid://130820223023461',swedishkronasignCircleFill=
'rbxassetid://96231802699012',swedishkronasignGaugeChartLefthalfRighthalf=
'rbxassetid://90449849273258',
swedishkronasignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://85748624360232',swedishkronasignRing=
'rbxassetid://105958553353498',swedishkronasignRingDashed=
'rbxassetid://115761962010012',swedishkronasignSquare=
'rbxassetid://128931505796379',swedishkronasignSquareFill=
'rbxassetid://86509975102581',swift='rbxassetid://135681872987394',swiftdata=
'rbxassetid://74282041662791',swirlCircleRighthalfFilled=
'rbxassetid://103405176082247',swirlCircleRighthalfFilledInverse=
'rbxassetid://75696463542233',switch2='rbxassetid://99218016450501',
switchProgrammable='rbxassetid://84251494436351',switchProgrammableFill=
'rbxassetid://85826821975584',switchProgrammableSquare=
'rbxassetid://109202516688828',switchProgrammableSquareFill=
'rbxassetid://140038440376679',syringe='rbxassetid://134779716432867',
syringeFill='rbxassetid://105718946268724',tCircle=
'rbxassetid://113271024609284',tCircleFill='rbxassetid://119571720599814',
tSquare='rbxassetid://87546740989322',tSquareFill='rbxassetid://117186355627025'
,tableFurniture='rbxassetid://118809430961100',tableFurnitureFill=
'rbxassetid://128445581622810',tablecells='rbxassetid://85004798836302',
tablecellsBadgeEllipsis='rbxassetid://79889013681142',tablecellsFill=
'rbxassetid://109601246909544',tablecellsFillBadgeEllipsis=
'rbxassetid://129079091406121',tachometer='rbxassetid://100567385026177',tag=
'rbxassetid://104278823123794',tagCircle='rbxassetid://74061891188761',
tagCircleFill='rbxassetid://114556711951516',tagFill=
'rbxassetid://89746504735605',tagSlash='rbxassetid://76661632603924',
tagSlashFill='rbxassetid://109012633470146',tagSquare=
'rbxassetid://87621218946117',tagSquareFill='rbxassetid://114890035110337',
taillightFog='rbxassetid://82333208021035',taillightFogFill=
'rbxassetid://99709323035641',takeoutbagAndCupAndStraw=
'rbxassetid://135148907059856',takeoutbagAndCupAndStrawFill=
'rbxassetid://103318663213516',target='rbxassetid://84037566052726',teddybear=
'rbxassetid://89621151313364',teddybearFill='rbxassetid://81297274380624',
teletype='rbxassetid://107362767762739',teletypeAnswer=
'rbxassetid://139631101200787',teletypeAnswerCircle=
'rbxassetid://90200447484716',teletypeAnswerCircleFill=
'rbxassetid://111381178181644',teletypeCircle='rbxassetid://138872701729896',
teletypeCircleFill='rbxassetid://80115468549244',tengesign=
'rbxassetid://76988463911430',tengesignArrowTriangleheadCounterclockwiseRotate90
='rbxassetid://75892461467217',tengesignBankBuilding=
'rbxassetid://112391984014443',tengesignBankBuildingFill=
'rbxassetid://126608973721961',tengesignCircle='rbxassetid://111459092131195',
tengesignCircleFill='rbxassetid://123410462749809',
tengesignGaugeChartLefthalfRighthalf='rbxassetid://95846595281122',
tengesignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://133479365260141',
tengesignRing='rbxassetid://134859673001983',tengesignRingDashed=
'rbxassetid://118456008853764',tengesignSquare='rbxassetid://108744664506456',
tengesignSquareFill='rbxassetid://137042676059462',tennisRacket=
'rbxassetid://117271664113564',tennisRacketCircle='rbxassetid://76019214542356',
tennisRacketCircleFill='rbxassetid://125737548729214',tennisball=
'rbxassetid://104744228251964',tennisballCircle='rbxassetid://121952139494198',
tennisballCircleFill='rbxassetid://75309517708716',tennisballFill=
'rbxassetid://94298365160647',tent='rbxassetid://107006023598958',tent2=
'rbxassetid://95982406617723',tent2Circle='rbxassetid://131199860472116',
tent2CircleFill='rbxassetid://85668483723444',tent2Fill=
'rbxassetid://91088067443876',tentCircle='rbxassetid://70607897797397',
tentCircleFill='rbxassetid://135778310997601',tentFill=
'rbxassetid://71561855929740',testtube2='rbxassetid://80066702459692',
textAligncenter='rbxassetid://71237993847541',textAlignleft=
'rbxassetid://126414082317242',textAlignright='rbxassetid://84350767056630',
textAndCommandMacwindow='rbxassetid://86957010693702',textAppend=
'rbxassetid://108335281304146',textBadgeCheckmark='rbxassetid://105812612281248'
,textBadgeMinus='rbxassetid://83327763553474',textBadgePlus=
'rbxassetid://129239148873683',textBadgeStar='rbxassetid://128027608333717',
textBadgeXmark='rbxassetid://101988186707064',textBelowFolder=
'rbxassetid://87822055347956',textBelowFolderFill='rbxassetid://130066865275998'
,textBelowPhoto='rbxassetid://139379154126272',textBelowPhotoFill=
'rbxassetid://134160367064989',textBookClosed='rbxassetid://111192629904131',
textBookClosedFill='rbxassetid://97841285729388',textBubble=
'rbxassetid://128793402405217',textBubbleBadgeClock=
'rbxassetid://93907819124534',textBubbleBadgeClockFill=
'rbxassetid://88289318147358',textBubbleFill='rbxassetid://116249834530262',
textDocument='rbxassetid://124309426221022',textDocumentFill=
'rbxassetid://124730277333625',textInsert='rbxassetid://97126037596934',
textJustify='rbxassetid://92189821886998',textJustifyLeading=
'rbxassetid://99075460859575',textJustifyLeft='rbxassetid://131541181777984',
textJustifyRight='rbxassetid://129280972878245',textJustifyTrailing=
'rbxassetid://129652399217283',textLine2Summary='rbxassetid://140230863430130',
textLine2SummaryBadgeXmark='rbxassetid://70429489437963',textLine3Summary=
'rbxassetid://81766828169006',textLineFirstAndArrowtriangleForward=
'rbxassetid://85328963821966',textLineLastAndArrowtriangleForward=
'rbxassetid://108463662081854',textLineMagnify='rbxassetid://95577209625603',
textMagnifyingglass='rbxassetid://114116769689132',textPadHeader=
'rbxassetid://106064080795897',textPadHeaderBadgeClock=
'rbxassetid://124262662260972',textPadHeaderBadgePlus=
'rbxassetid://136283903785863',textPage='rbxassetid://92314254916759',
textPageBadgeMagnifyingglass='rbxassetid://127124929104557',textPageFill=
'rbxassetid://139449259721285',textPageSlash='rbxassetid://113189115705323',
textPageSlashFill='rbxassetid://105274129885608',textQuote=
'rbxassetid://71027928151952',textRectangle='rbxassetid://130318963677643',
textRectangleFill='rbxassetid://108940510434765',textRectanglePage=
'rbxassetid://86723219970269',textRectanglePageFill=
'rbxassetid://114057346576950',textRedaction='rbxassetid://105235021121957',
textSquareFilled='rbxassetid://112727132354588',textViewfinder=
'rbxassetid://95798719444148',textWordSpacing='rbxassetid://75789106006521',
textformat='rbxassetid://107710559061740',textformatAlt=
'rbxassetid://132159623766191',textformatCharacters=
'rbxassetid://117802849867936',textformatCharactersArrowLeftAndRight=
'rbxassetid://129388475589128',textformatCharactersDottedunderline=
'rbxassetid://78315218057352',textformatNumbers='rbxassetid://124177082221864',
textformatSize='rbxassetid://93771567593788',textformatSizeLarger=
'rbxassetid://93387252147805',textformatSizeSmaller=
'rbxassetid://109995000982679',textformatSubscript='rbxassetid://86381798212299'
,textformatSuperscript='rbxassetid://81076327481935',theatermaskAndPaintbrush=
'rbxassetid://88404865436421',theatermaskAndPaintbrushFill=
'rbxassetid://127636624966411',theatermasks='rbxassetid://119325864262704',
theatermasksCircle='rbxassetid://80160373236658',theatermasksCircleFill=
'rbxassetid://122778091246019',theatermasksFill='rbxassetid://118820244647209',
thermometerAndEllipsis='rbxassetid://106404759350042',thermometerAndLiquidWaves=
'rbxassetid://110474674863751',thermometerAndLiquidWavesSnowflake=
'rbxassetid://127007809849613',
thermometerAndLiquidWavesTrianglebadgeExclamationmark=
'rbxassetid://74515683264588',thermometerBrakesignal=
'rbxassetid://78708806961918',thermometerGaugeOpen=
'rbxassetid://122264305417542',thermometerHigh='rbxassetid://89229451444519',
thermometerLow='rbxassetid://131420426530700',thermometerMedium=
'rbxassetid://71148830790297',thermometerMediumSlash=
'rbxassetid://118717870879578',thermometerSnowflake=
'rbxassetid://75391106253368',thermometerSnowflakeCircle=
'rbxassetid://131952215879048',thermometerSnowflakeCircleFill=
'rbxassetid://133994251139774',thermometerSun='rbxassetid://107951087495619',
thermometerSunCircle='rbxassetid://82549716179412',thermometerSunCircleFill=
'rbxassetid://74970074280056',thermometerSunFill='rbxassetid://120808413699971',
thermometerTirepressure='rbxassetid://89391830630971',thermometerTransmission=
'rbxassetid://84090975599959',thermometerVariable='rbxassetid://123361025458437'
,thermometerVariableAndFigure='rbxassetid://71146322185292',
thermometerVariableAndFigureCircle='rbxassetid://126503466826501',
thermometerVariableAndFigureCircleFill='rbxassetid://112639012001535',
thermometerVariableBadgeClock='rbxassetid://77637851179488',
thermometerVariableBadgePlay='rbxassetid://138406770912999',ticket=
'rbxassetid://78850938480744',ticketCircle='rbxassetid://125548492810962',
ticketCircleFill='rbxassetid://81789279996088',ticketFill=
'rbxassetid://134666026768258',timelapse='rbxassetid://96365441062279',
timelineSelection='rbxassetid://93679306913299',timer=
'rbxassetid://131232244017070',timerCircle='rbxassetid://104963768880474',
timerCircleFill='rbxassetid://74781722205278',timerSquare=
'rbxassetid://136926278456877',tire='rbxassetid://121804115558084',
tireBadgeSnowflake='rbxassetid://118870916845492',tirepressure=
'rbxassetid://139354888430441',togglepower='rbxassetid://126792360918660',toilet
='rbxassetid://139322044959527',toiletCircle='rbxassetid://132862220807496',
toiletCircleFill='rbxassetid://86592804374521',toiletFill=
'rbxassetid://89608461240931',tornado='rbxassetid://93394540448953',
tornadoCircle='rbxassetid://120622162002991',tornadoCircleFill=
'rbxassetid://119975169838498',tortoise='rbxassetid://89207957369788',
tortoiseCircle='rbxassetid://138114227730926',tortoiseCircleFill=
'rbxassetid://113462643523462',tortoiseFill='rbxassetid://108775029902688',torus
='rbxassetid://132235380088261',touchid='rbxassetid://127151466748359',towHitch=
'rbxassetid://98604376318167',towHitchExclamationmark=
'rbxassetid://117587622340065',towHitchExclamationmarkFill=
'rbxassetid://77885472071490',towHitchFill='rbxassetid://136757672238841',
tractionControlTirepressure='rbxassetid://89446879516569',
tractionControlTirepressureExclamationmark='rbxassetid://129579264199158',
tractionControlTirepressureSlash='rbxassetid://82258262400115',trainSideFrontCar
='rbxassetid://81352707597032',trainSideMiddleCar='rbxassetid://84241439668483',
trainSideRearCar='rbxassetid://114688400647868',tram=
'rbxassetid://84554772135786',tramCard='rbxassetid://94311647704427',
tramCardFill='rbxassetid://72197519771173',tramCircle=
'rbxassetid://99917178213901',tramCircleFill='rbxassetid://84107263038489',
tramFill='rbxassetid://124882868874709',tramFillTunnel=
'rbxassetid://82427435127609',translate='rbxassetid://74057651757937',
transmission='rbxassetid://78486925087835',trapezoidAndLineHorizontal=
'rbxassetid://117248222414382',trapezoidAndLineHorizontalFill=
'rbxassetid://83400087360136',trapezoidAndLineVertical=
'rbxassetid://122849535680037',trapezoidAndLineVerticalFill=
'rbxassetid://83143223589474',trash='rbxassetid://138210305619993',trashCircle=
'rbxassetid://73759030470557',trashCircleFill='rbxassetid://108682997886275',
trashFill='rbxassetid://106522682727620',trashSlash=
'rbxassetid://137129524605524',trashSlashCircle='rbxassetid://75985404698667',
trashSlashCircleFill='rbxassetid://119475662673602',trashSlashFill=
'rbxassetid://86415395411354',trashSlashSquare='rbxassetid://75415733626570',
trashSlashSquareFill='rbxassetid://111819411879473',trashSquare=
'rbxassetid://139489659801239',trashSquareFill='rbxassetid://109042030275157',
tray='rbxassetid://73132187606290',tray2='rbxassetid://109273606771685',
tray2Fill='rbxassetid://114990115453763',trayAndArrowDown=
'rbxassetid://138530153829354',trayAndArrowDownFill=
'rbxassetid://91720907960301',trayAndArrowUp='rbxassetid://77418052891963',
trayAndArrowUpFill='rbxassetid://111348199178010',trayBadge=
'rbxassetid://99876442266432',trayBadgeFill='rbxassetid://101926737161042',
trayCircle='rbxassetid://98784166534236',trayCircleFill=
'rbxassetid://131639114232091',trayFill='rbxassetid://122084538861944',trayFull=
'rbxassetid://132151007573012',trayFullFill='rbxassetid://91743715077166',tree=
'rbxassetid://131434292632259',treeCircle='rbxassetid://77869416588663',
treeCircleFill='rbxassetid://118871328720111',treeFill=
'rbxassetid://127929585101588',triangle='rbxassetid://103147443822886',
triangleBottomhalfFilled='rbxassetid://91709533626215',triangleCircle=
'rbxassetid://111397146609147',triangleCircleFill='rbxassetid://131784417798929'
,triangleFill='rbxassetid://98911778107051',triangleLefthalfFilled=
'rbxassetid://122088513245439',triangleRighthalfFilled=
'rbxassetid://133484476485466',triangleTophalfFilled=
'rbxassetid://114334455043672',triangleshape='rbxassetid://103860060139536',
triangleshapeFill='rbxassetid://137360750563384',trophy=
'rbxassetid://113423520493674',trophyCircle='rbxassetid://80943025177785',
trophyCircleFill='rbxassetid://76982971562787',trophyFill=
'rbxassetid://74977361750490',tropicalstorm='rbxassetid://97305087052372',
tropicalstormCircle='rbxassetid://117783495371346',tropicalstormCircleFill=
'rbxassetid://90338028011988',truckBox='rbxassetid://83851227799136',
truckBoxBadgeClock='rbxassetid://89308426660825',truckBoxBadgeClockFill=
'rbxassetid://132276325552498',truckBoxFill='rbxassetid://110475724637231',
truckPickupSide='rbxassetid://123433678430058',truckPickupSideAirCirculate=
'rbxassetid://80768683207277',truckPickupSideAirCirculateFill=
'rbxassetid://101108911081174',truckPickupSideAirFresh=
'rbxassetid://71635746134403',truckPickupSideAirFreshFill=
'rbxassetid://87057182945703',truckPickupSideAndExclamationmark=
'rbxassetid://139134957787137',truckPickupSideAndExclamationmarkFill=
'rbxassetid://95516163286881',truckPickupSideArrowLeftAndRight=
'rbxassetid://134277371887972',truckPickupSideArrowLeftAndRightFill=
'rbxassetid://120454836734609',truckPickupSideArrowtriangleDown=
'rbxassetid://132765732566337',truckPickupSideArrowtriangleDownFill=
'rbxassetid://126435076757533',truckPickupSideArrowtriangleUp=
'rbxassetid://129927853101620',truckPickupSideArrowtriangleUpArrowtriangleDown=
'rbxassetid://76463006668039',
truckPickupSideArrowtriangleUpArrowtriangleDownFill=
'rbxassetid://102914968611680',truckPickupSideArrowtriangleUpFill=
'rbxassetid://103734412692534',truckPickupSideFill='rbxassetid://97841418868342'
,truckPickupSideFrontOpen='rbxassetid://130338401662393',
truckPickupSideFrontOpenCrop='rbxassetid://130602145974071',
truckPickupSideFrontOpenCropFill='rbxassetid://80394069134521',
truckPickupSideFrontOpenFill='rbxassetid://140564339078125',
truckPickupSideHillDown='rbxassetid://81855572295261',
truckPickupSideHillDownAndGaugeOpenWithLinesNeedle25percentAndArrowtriangle=
'rbxassetid://120535414775272',
truckPickupSideHillDownAndGaugeOpenWithLinesNeedle25percentAndArrowtriangleFill=
'rbxassetid://96717003085498',truckPickupSideHillDownFill=
'rbxassetid://129981288263838',truckPickupSideHillUp=
'rbxassetid://120531308583244',truckPickupSideHillUpFill=
'rbxassetid://118462546162839',truckPickupSideLock='rbxassetid://90874238611008'
,truckPickupSideLockFill='rbxassetid://139300624211362',truckPickupSideLockOpen=
'rbxassetid://111751710170128',truckPickupSideLockOpenFill=
'rbxassetid://117854506670260',truckSideHillDescentControl=
'rbxassetid://106358378146490',truckSideHillDescentControlFill=
'rbxassetid://100686764237291',truckSideRoofCargoCarrier=
'rbxassetid://71548640889981',truckSideRoofCargoCarrierFill=
'rbxassetid://70901858186322',truckSideRoofCargoCarrierSlash=
'rbxassetid://107972634370880',truckSideRoofCargoCarrierSlashFill=
'rbxassetid://122020656964646',tsa='rbxassetid://134948426765455',tsaCircle=
'rbxassetid://108199751556422',tsaCircleFill='rbxassetid://72935539681008',
tsaSlash='rbxassetid://99849566895544',tshirt='rbxassetid://78149665788013',
tshirtCircle='rbxassetid://124062591825671',tshirtCircleFill=
'rbxassetid://95738553763485',tshirtFill='rbxassetid://93980224214143',
tugriksign='rbxassetid://81138137513450',
tugriksignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://126226333785383',tugriksignBankBuilding=
'rbxassetid://110652295230466',tugriksignBankBuildingFill=
'rbxassetid://109410303282690',tugriksignCircle='rbxassetid://100568170639157',
tugriksignCircleFill='rbxassetid://86051072080818',
tugriksignGaugeChartLefthalfRighthalf='rbxassetid://122980287842808',
tugriksignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://71970544968459',
tugriksignRing='rbxassetid://114551163304551',tugriksignRingDashed=
'rbxassetid://131188971913163',tugriksignSquare='rbxassetid://113572701477700',
tugriksignSquareFill='rbxassetid://102159003271983',tuningfork=
'rbxassetid://101642064626347',turkishlirasign='rbxassetid://100432595691389',
turkishlirasignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://125963477298669',turkishlirasignBankBuilding=
'rbxassetid://86988859548682',turkishlirasignBankBuildingFill=
'rbxassetid://101316166861444',turkishlirasignCircle=
'rbxassetid://129136113627430',turkishlirasignCircleFill=
'rbxassetid://96077985019311',turkishlirasignGaugeChartLefthalfRighthalf=
'rbxassetid://132308066882364',
turkishlirasignGaugeChartLeftthirdTopthirdRightthird=
'rbxassetid://135569336786362',turkishlirasignRing=
'rbxassetid://126842663496567',turkishlirasignRingDashed=
'rbxassetid://89470219812321',turkishlirasignSquare=
'rbxassetid://80810143950534',turkishlirasignSquareFill=
'rbxassetid://81730250023213',tv='rbxassetid://134683634154697',
tvAndHifispeakerFill='rbxassetid://126365816064099',tvAndMediabox=
'rbxassetid://86190133697909',tvAndMediaboxFill='rbxassetid://70794673467646',
tvBadgeWifi='rbxassetid://134114499802296',tvBadgeWifiFill=
'rbxassetid://99921301995389',tvCircle='rbxassetid://108467456786283',
tvCircleFill='rbxassetid://104378375131972',tvFill=
'rbxassetid://134052066796282',tvSlash='rbxassetid://87329837865736',tvSlashFill
='rbxassetid://138613370587055',uCircle='rbxassetid://133864802523646',
uCircleFill='rbxassetid://83802229447704',uSquare='rbxassetid://70478293186949',
uSquareFill='rbxassetid://103098702036404',uiwindowSplit2x1=
'rbxassetid://94654002409260',umbrella='rbxassetid://107688268920202',
umbrellaCircle='rbxassetid://78176157826476',umbrellaCircleFill=
'rbxassetid://90379909796237',umbrellaFill='rbxassetid://125319557780841',
umbrellaGaugeOpen='rbxassetid://115390159124102',umbrellaPercent=
'rbxassetid://74004447484301',umbrellaPercentFill='rbxassetid://98934968468500',
umbrellaSensorTagRadiowavesLeftAndRight='rbxassetid://111616368640765',
umbrellaSensorTagRadiowavesLeftAndRightFill='rbxassetid://84410220519393',
underline='rbxassetid://79716535798546',underlineDouble=
'rbxassetid://95042272345021',vCircle='rbxassetid://78905420701975',vCircleFill=
'rbxassetid://109231824858777',vSquare='rbxassetid://112544504091047',
vSquareFill='rbxassetid://95651976776858',ventHeatWavesUpward=
'rbxassetid://81373110761347',vialViewfinder='rbxassetid://120653091554663',
video='rbxassetid://75234641923007',videoBadgeCheckmark=
'rbxassetid://93588020970649',videoBadgeEllipsis='rbxassetid://73931355581093',
videoBadgePlus='rbxassetid://77189160054067',videoBadgeWaveform=
'rbxassetid://94770313707904',videoBadgeWaveformFill=
'rbxassetid://91311029131545',videoBubble='rbxassetid://111260570124310',
videoBubbleFill='rbxassetid://98571132429169',videoCircle=
'rbxassetid://126991025094281',videoCircleFill='rbxassetid://102639060158177',
videoDoorbell='rbxassetid://127205026449016',videoDoorbellFill=
'rbxassetid://87815257505023',videoFill='rbxassetid://139148806818337',
videoFillBadgeCheckmark='rbxassetid://94758158365454',videoFillBadgeEllipsis=
'rbxassetid://115997809417535',videoFillBadgePlus='rbxassetid://71066361916528',
videoSlash='rbxassetid://77940555638934',videoSlashCircle=
'rbxassetid://78499840932408',videoSlashCircleFill=
'rbxassetid://106447121815942',videoSlashFill='rbxassetid://85954124688252',
videoSquare='rbxassetid://74932084950455',videoSquareFill=
'rbxassetid://108239592630600',videoprojector='rbxassetid://102503253637538',
videoprojectorFill='rbxassetid://123004409391173',view2d=
'rbxassetid://137344398414768',view3d='rbxassetid://108397171176671',viewfinder=
'rbxassetid://94783896988587',viewfinderCircle='rbxassetid://121365827431194',
viewfinderCircleFill='rbxassetid://123480648427476',viewfinderRectangular=
'rbxassetid://92938228120219',viewfinderTrianglebadgeExclamationmark=
'rbxassetid://110389074265828',visionPro='rbxassetid://86658888521866',
visionProAndArrowForward='rbxassetid://92640289852602',
visionProAndArrowForwardFill='rbxassetid://131506122489642',
visionProBadgeCheckmark='rbxassetid://91300537956331',
visionProBadgeCheckmarkFill='rbxassetid://131430228822693',
visionProBadgeExclamationmark='rbxassetid://131951123332852',
visionProBadgeExclamationmarkFill='rbxassetid://127300971189191',
visionProBadgePlay='rbxassetid://86395593898709',visionProBadgePlayFill=
'rbxassetid://97801624322845',visionProCircle='rbxassetid://97576563076177',
visionProCircleFill='rbxassetid://139552674437388',visionProFill=
'rbxassetid://108432448592279',visionProSlash='rbxassetid://109140558729897',
visionProSlashCircle='rbxassetid://88113051707571',visionProSlashCircleFill=
'rbxassetid://135642237204795',visionProSlashFill='rbxassetid://121260021824503'
,visionProTrianglebadgeExclamationmark='rbxassetid://134894440127646',
visionProTrianglebadgeExclamationmarkFill='rbxassetid://91237961536763',
voiceover='rbxassetid://136800595376174',volleyball=
'rbxassetid://73970433297176',volleyballCircle='rbxassetid://76121885043459',
volleyballCircleFill='rbxassetid://139162144058450',volleyballFill=
'rbxassetid://78657080282621',wCircle='rbxassetid://83697484178752',wCircleFill=
'rbxassetid://82337186361293',wSquare='rbxassetid://71979974267181',wSquareFill=
'rbxassetid://81077708796392',wake='rbxassetid://134066791044326',wakeCircle=
'rbxassetid://96591606910274',wakeCircleFill='rbxassetid://79093975624302',
walletBifold='rbxassetid://136364251846065',walletBifoldFill=
'rbxassetid://119240717399847',walletPass='rbxassetid://134909953531837',
walletPassFill='rbxassetid://96808145241944',
walletSensorTagRadiowavesLeftAndRight='rbxassetid://126915780100901',
walletSensorTagRadiowavesLeftAndRightFill='rbxassetid://81271116607581',
wandAndOutline='rbxassetid://129284003313828',wandAndOutlineInverse=
'rbxassetid://71138507542324',wandAndRays='rbxassetid://92752161512332',
wandAndRaysInverse='rbxassetid://125975271070513',wandAndSparkles=
'rbxassetid://72040453503595',wandAndSparklesInverse=
'rbxassetid://79333469702486',warninglight='rbxassetid://84007946028451',
warninglightFill='rbxassetid://109840072719326',washer=
'rbxassetid://93685448107607',washerCircle='rbxassetid://105378880332708',
washerCircleFill='rbxassetid://119955243167699',washerFill=
'rbxassetid://89945593257514',watchAnalog='rbxassetid://88617234755283',
watchfaceApplewatchCase='rbxassetid://73863272801256',waterWaves=
'rbxassetid://138305668238725',waterWavesAndArrowTriangleheadDown=
'rbxassetid://90118761233028',
waterWavesAndArrowTriangleheadDownTrianglebadgeExclamationmark=
'rbxassetid://85387752450910',waterWavesAndArrowTriangleheadUp=
'rbxassetid://127164136396826',waterWavesSlash='rbxassetid://111588668015905',
waterbottle='rbxassetid://138004611200910',waterbottleFill=
'rbxassetid://121256533818085',wave3Backward='rbxassetid://80847645712657',
wave3BackwardCircle='rbxassetid://136031088324218',wave3BackwardCircleFill=
'rbxassetid://71061533127513',wave3Down='rbxassetid://108832060215003',
wave3DownCarSide='rbxassetid://123655331264678',wave3DownCarSideFill=
'rbxassetid://115209382072100',wave3DownCircle='rbxassetid://114336657044130',
wave3DownCircleFill='rbxassetid://74329989717731',wave3DownConvertibleSide=
'rbxassetid://131401811268042',wave3DownConvertibleSideFill=
'rbxassetid://79499527400603',wave3DownPickupSide='rbxassetid://97436480627142',
wave3DownPickupSideFill='rbxassetid://129019641239034',wave3DownSuvSide=
'rbxassetid://71035812422373',wave3DownSuvSideFill=
'rbxassetid://134610119907297',wave3Forward='rbxassetid://111357002729843',
wave3ForwardCircle='rbxassetid://80066074333627',wave3ForwardCircleFill=
'rbxassetid://80719773619963',wave3Left='rbxassetid://117436129898300',
wave3LeftCircle='rbxassetid://85305739132620',wave3LeftCircleFill=
'rbxassetid://93517609770045',wave3Right='rbxassetid://92963290455418',
wave3RightCircle='rbxassetid://91349038271768',wave3RightCircleFill=
'rbxassetid://140061124860963',wave3Up='rbxassetid://128528619550054',
wave3UpCircle='rbxassetid://133995083701260',wave3UpCircleFill=
'rbxassetid://96665619623184',waveform='rbxassetid://136809466139252',
waveformAndPersonFilled='rbxassetid://126270512457294',waveformBadgeCheckmark=
'rbxassetid://128595421098352',waveformBadgeExclamationmark=
'rbxassetid://120204943860124',waveformBadgeMagnifyingglass=
'rbxassetid://121985661181170',waveformBadgeMicrophone=
'rbxassetid://87232803530574',waveformBadgeMinus='rbxassetid://102741667650470',
waveformBadgePlus='rbxassetid://102403202693811',waveformBadgeXmark=
'rbxassetid://120155576829911',waveformCircle='rbxassetid://84018574438696',
waveformCircleFill='rbxassetid://98502737702633',waveformLow=
'rbxassetid://95721363422192',waveformMid='rbxassetid://73459359791943',
waveformPath='rbxassetid://72307154936317',waveformPathBadgeMinus=
'rbxassetid://136232811740839',waveformPathBadgePlus=
'rbxassetid://127759512381534',waveformPathEcg='rbxassetid://128118780008401',
waveformPathEcgMagnifyingglass='rbxassetid://79598417522204',
waveformPathEcgRectangle='rbxassetid://70398715649395',
waveformPathEcgRectangleFill='rbxassetid://90914728496991',waveformPathEcgText=
'rbxassetid://116861166666873',waveformPathEcgTextClipboard=
'rbxassetid://81460112831245',waveformPathEcgTextClipboardFill=
'rbxassetid://130425262607160',waveformPathEcgTextPage=
'rbxassetid://101998995563939',waveformPathEcgTextPageFill=
'rbxassetid://85336259615594',waveformSlash='rbxassetid://101318589553195',
webCamera='rbxassetid://85276511025498',webCameraFill=
'rbxassetid://86988545799263',wheelchair='rbxassetid://130234014770883',
widgetExtralarge='rbxassetid://87179649600036',widgetExtralargeBadgePlus=
'rbxassetid://87961000171364',widgetLarge='rbxassetid://138302819665976',
widgetLargeBadgePlus='rbxassetid://102309628278255',widgetMedium=
'rbxassetid://123766243611909',widgetMediumBadgePlus=
'rbxassetid://129502000388063',widgetSmall='rbxassetid://133222809195444',
widgetSmallBadgePlus='rbxassetid://106813307987533',wifi=
'rbxassetid://127365451355653',wifiBadgeLock='rbxassetid://105865344224961',
wifiCircle='rbxassetid://131659272048007',wifiCircleFill=
'rbxassetid://123823896793006',wifiExclamationmark=
'rbxassetid://117799256745502',wifiExclamationmarkCircle=
'rbxassetid://133542793616716',wifiExclamationmarkCircleFill=
'rbxassetid://103666079404717',wifiRouter='rbxassetid://107283421485159',
wifiRouterFill='rbxassetid://127625180753277',wifiSlash=
'rbxassetid://105782001663907',wifiSquare='rbxassetid://101097897107565',
wifiSquareFill='rbxassetid://135210787558367',wind='rbxassetid://80700050884353'
,windCircle='rbxassetid://129546295733216',windCircleFill=
'rbxassetid://96079739586089',windSnow='rbxassetid://92339772641709',
windSnowCircle='rbxassetid://111653489867204',windSnowCircleFill=
'rbxassetid://132514124832058',windowAwning='rbxassetid://136043702603149',
windowAwningClosed='rbxassetid://75144791958276',windowCasement=
'rbxassetid://75282400690778',windowCasementClosed=
'rbxassetid://118468264534240',windowCeiling='rbxassetid://120918397527890',
windowCeilingClosed='rbxassetid://72533185677487',windowHorizontal=
'rbxassetid://92196780376848',windowHorizontalClosed=
'rbxassetid://77912685751805',windowShadeClosed='rbxassetid://99464758323012',
windowShadeOpen='rbxassetid://74340846109012',windowVerticalClosed=
'rbxassetid://77549144479343',windowVerticalOpen='rbxassetid://116989707350034',
windshieldFrontAndFluidAndSpray='rbxassetid://104140821821775',
windshieldFrontAndHeatWaves='rbxassetid://139231996805155',
windshieldFrontAndSpray='rbxassetid://118383554311432',windshieldFrontAndWiper=
'rbxassetid://90221744279393',windshieldFrontAndWiperAndDrop=
'rbxassetid://84550752400798',windshieldFrontAndWiperAndSpray=
'rbxassetid://127696811498029',windshieldFrontAndWiperExclamationmark=
'rbxassetid://113077812388437',windshieldFrontAndWiperIntermittent=
'rbxassetid://130126356387267',windshieldRearAndFluidAndSpray=
'rbxassetid://71902151124766',windshieldRearAndHeatWaves=
'rbxassetid://85175322908644',windshieldRearAndSpray=
'rbxassetid://74325024162387',windshieldRearAndWiper=
'rbxassetid://132496191383598',windshieldRearAndWiperAndDrop=
'rbxassetid://91997491758687',windshieldRearAndWiperAndSpray=
'rbxassetid://70577185858512',windshieldRearAndWiperExclamationmark=
'rbxassetid://103508258069471',windshieldRearAndWiperIntermittent=
'rbxassetid://112006728415524',wineglass='rbxassetid://101074630979300',
wineglassFill='rbxassetid://114802765580227',wonsign=
'rbxassetid://138881045027066',wonsignArrowTriangleheadCounterclockwiseRotate90=
'rbxassetid://127138253634243',wonsignBankBuilding=
'rbxassetid://111293124675993',wonsignBankBuildingFill=
'rbxassetid://84415197244225',wonsignCircle='rbxassetid://88823142706925',
wonsignCircleFill='rbxassetid://137966197891409',
wonsignGaugeChartLefthalfRighthalf='rbxassetid://111492435126131',
wonsignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://121709795250122',
wonsignRing='rbxassetid://76574048523234',wonsignRingDashed=
'rbxassetid://133779937082970',wonsignSquare='rbxassetid://132391856360168',
wonsignSquareFill='rbxassetid://90329995457020',wrenchAdjustable=
'rbxassetid://136627575678034',wrenchAdjustableFill=
'rbxassetid://113988495428000',wrenchAndScrewdriver=
'rbxassetid://103452076936452',wrenchAndScrewdriverFill=
'rbxassetid://105571024016922',wrongwaysign='rbxassetid://76431850074308',
wrongwaysignFill='rbxassetid://83735880477861',xCircle=
'rbxassetid://91981193078729',xCircleFill='rbxassetid://94299891039084',xSquare=
'rbxassetid://114578513610753',xSquareFill='rbxassetid://111170360358521',
xSquareroot='rbxassetid://78470545377271',xboxLogo='rbxassetid://89705591865004'
,xmark='rbxassetid://80129517509086',xmarkApp='rbxassetid://77614287062581',
xmarkAppFill='rbxassetid://70913517985061',xmarkBin=
'rbxassetid://115092753402225',xmarkBinCircle='rbxassetid://100527660496479',
xmarkBinCircleFill='rbxassetid://72818314748638',xmarkBinFill=
'rbxassetid://93339862156011',xmarkCircle='rbxassetid://74086380322275',
xmarkCircleBadgeAirplane='rbxassetid://107048545652410',
xmarkCircleBadgeAirplaneFill='rbxassetid://107260850906533',xmarkCircleFill=
'rbxassetid://76720364341917',xmarkDiamond='rbxassetid://123566993050169',
xmarkDiamondFill='rbxassetid://93992936564694',xmarkIcloud=
'rbxassetid://94493472046695',xmarkIcloudFill='rbxassetid://76302466621348',
xmarkOctagon='rbxassetid://91338686100699',xmarkOctagonFill=
'rbxassetid://90341839195452',xmarkRectangle='rbxassetid://121990685238111',
xmarkRectangleFill='rbxassetid://74076416348699',xmarkRectanglePortrait=
'rbxassetid://76998319704304',xmarkRectanglePortraitFill=
'rbxassetid://108883994990659',xmarkSeal='rbxassetid://135175848462028',
xmarkSealFill='rbxassetid://138857703888143',xmarkShield=
'rbxassetid://112602820350263',xmarkShieldFill='rbxassetid://129190191318804',
xmarkSquare='rbxassetid://94780384109918',xmarkSquareFill=
'rbxassetid://113379805795943',xmarkTriangleCircleSquare=
'rbxassetid://93482352448078',xmarkTriangleCircleSquareFill=
'rbxassetid://92930583604823',xserve='rbxassetid://100936689010686',xserveRaid=
'rbxassetid://96204733517915',yCircle='rbxassetid://104458771421352',yCircleFill
='rbxassetid://82033787913749',ySquare='rbxassetid://103183886841769',
ySquareFill='rbxassetid://72405810989202',yensign='rbxassetid://99734954362670',
yensignArrowTriangleheadCounterclockwiseRotate90='rbxassetid://80201233274715',
yensignBankBuilding='rbxassetid://72141299288880',yensignBankBuildingFill=
'rbxassetid://110179200008310',yensignCircle='rbxassetid://108854048682589',
yensignCircleFill='rbxassetid://112746170982965',
yensignGaugeChartLefthalfRighthalf='rbxassetid://115445828866168',
yensignGaugeChartLeftthirdTopthirdRightthird='rbxassetid://90119523470045',
yensignRing='rbxassetid://80789867463387',yensignRingDashed=
'rbxassetid://78763231940715',yensignSquare='rbxassetid://124557047246185',
yensignSquareFill='rbxassetid://96601254376112',yieldsign=
'rbxassetid://80696636377723',yieldsignFill='rbxassetid://99571675480041',
zCircle='rbxassetid://122572624815216',zCircleFill='rbxassetid://79133738654460'
,zSquare='rbxassetid://96274174435903',zSquareFill='rbxassetid://91137135507817'
,zipperPage='rbxassetid://74105406352571',zlButtonRoundedtopHorizontal=
'rbxassetid://92649509158613',zlButtonRoundedtopHorizontalFill=
'rbxassetid://88379112766255',zrButtonRoundedtopHorizontal=
'rbxassetid://126371346858487',zrButtonRoundedtopHorizontalFill=
'rbxassetid://95978592996094',zzz='rbxassetid://81661835392914',number0Circle=
'rbxassetid://112630144563845',note='rbxassetid://93279801394323',number10Square
='rbxassetid://81808119537499',doc='rbxassetid://80680488161934',
number6AltSquare='rbxassetid://133921705918574',letterNCircle=
'rbxassetid://108501754774630',cursorarrowMotionlinesClick=
'rbxassetid://81573780818011',number26Square='rbxassetid://72769891775076',
letterBSquare='rbxassetid://107926268990493',squareshapeDottedSplit2x2=
'rbxassetid://91166447813616',
figureSeatedSideWindshieldFrontAndHeatWavesAirDistributionMiddleAndLower=
'rbxassetid://95139994219690',letterRJoystickPressDown=
'rbxassetid://101313826207669',number50Square='rbxassetid://89126185617207',
numeric4hCircle='rbxassetid://108759735497325',letterACircle=
'rbxassetid://102043519807680',flagCheckered2Crossed=
'rbxassetid://78412090520637',cursorarrowSlash='rbxassetid://91144329352020',
number0Square='rbxassetid://83690626547054',docZipper=
'rbxassetid://96676285483415',numeric2h='rbxassetid://105497397405088',
gobackward30='rbxassetid://140354039230051',letterKCircle=
'rbxassetid://126766755298208',textformatAbc='rbxassetid://138305677898152',
shieldCheckered='rbxassetid://122359446295344',beatsFitProRight=
'rbxassetid://119699746025879',
figureSeatedSideWindshieldFrontAndHeatWavesAirDistributionLower=
'rbxassetid://95211211764474',number44Circle='rbxassetid://95645523276973',
number1Circle='rbxassetid://90420045405973',beatsPowerbeatsproChargingcase=
'rbxassetid://129174227736582',squareshapeSplit2x2Dotted=
'rbxassetid://128554723537974',number19Square='rbxassetid://102272936095513',
number31Square='rbxassetid://72665219530870',cursorarrowClickBadgeClock=
'rbxassetid://96674815416136',letterCSquare='rbxassetid://77342805733080',
number4Square='rbxassetid://85342885629372',letterRButtonRoundedbottomHorizontal
='rbxassetid://127892690915857',figureSoccer='rbxassetid://116190117656300',
waterWavesAndArrowDownTrianglebadgeExclamationmark=
'rbxassetid://126530403017273',number9AltCircle='rbxassetid://89157942477413',
arrowTriangle2CirclepathCircle='rbxassetid://137167033413367',number12Lane=
'rbxassetid://122996912273287',letterDCircle='rbxassetid://77271013016863',
micCircle='rbxassetid://97798740260123',number3Lane=
'rbxassetid://123566814024217',cursorarrowClick='rbxassetid://120755931811218',
number09Circle='rbxassetid://76455988018714',number37Square=
'rbxassetid://111356313667440',number46Square='rbxassetid://71827970370485',
ipodshuffleGen2='rbxassetid://85111088101023',chartBarDocHorizontal=
'rbxassetid://88206449206082',textformat123='rbxassetid://134142879344254',
number9Square='rbxassetid://113640399773664',number23Circle=
'rbxassetid://113967161448579',letterLJoystickTiltDown=
'rbxassetid://109426620639396',number08Square='rbxassetid://128176821770996',
airpodproLeft='rbxassetid://95793086337265',beatsStudiobudsplusChargingcase=
'rbxassetid://125879752007426',textformatAbcDottedunderline=
'rbxassetid://94626317959618',arrowTriangleCapsulepath=
'rbxassetid://79935404561753',goforwardPlus='rbxassetid://130621369072851',
number22Square='rbxassetid://107808365980228',number07Square=
'rbxassetid://80554919883042',cursorarrowRays='rbxassetid://126392116415147',
headProfileArrowForwardAndVisionpro='rbxassetid://121693109509839',flagCheckered
='rbxassetid://78821229595515',number29Circle='rbxassetid://112151950673070',
letterXSquareroot='rbxassetid://80429022258285',number23Square=
'rbxassetid://126327561252667',letterZCircle='rbxassetid://109541227310226',
ipodshuffleGen3='rbxassetid://123523059376894',
figureSeatedSideWindshieldFrontAndHeatWavesAirDistributionMiddle=
'rbxassetid://79055808238898',
figureSeatedSideAirDistributionUpperAngledAndMiddleAndLowerAngled=
'rbxassetid://112235343241027',letterLSquare='rbxassetid://102129122454965',
letterMCircle='rbxassetid://105252616936701',figureSeatedSide=
'rbxassetid://103157176969588',rectangleInsetBadgeRecord=
'rbxassetid://80945153582789',number05Square='rbxassetid://114379850471491',
number28Circle='rbxassetid://77630558937328',number11Square=
'rbxassetid://137943364469277',letterYSquare='rbxassetid://80658772608290',
arrowTriangleMerge='rbxassetid://135675192910274',
pointBottomleftForwardToArrowtriangleUturnScurvepath=
'rbxassetid://137756040106336',letterISquare='rbxassetid://78227741894490',
figureSeatedSideWindshieldFrontAndHeatWavesAirDistributionUpperAndMiddleAndLower
='rbxassetid://133887310459410',figureSeatedSideAirbagOff2=
'rbxassetid://123574096899524',textformat12='rbxassetid://96656679869431',
numeric4a='rbxassetid://86756867568912',number02Circle=
'rbxassetid://133242670562001',figureSeatedSideAirbagOn=
'rbxassetid://112751749735103',arrowTriangle2CirclepathDocOnClipboard=
'rbxassetid://102874106989561',ipodshuffleGen1='rbxassetid://85956093711602',
beatsFitProLeft='rbxassetid://116602459616537',arrowDownDoc=
'rbxassetid://126263364298576',visionproBadgePlay='rbxassetid://122979325623328'
,waterWavesAndArrowUp='rbxassetid://117956742428464',
exclamationmarkArrowCirclepath='rbxassetid://111446640991646',
checkmarkGobackward='rbxassetid://137921605172447',waveformBadgeMic=
'rbxassetid://140630166066509',numeric4aCircle='rbxassetid://80949635570356',
beatsStudiobudsplusRight='rbxassetid://118741913661751',number01Square=
'rbxassetid://97241538934163',letterRJoystick='rbxassetid://77839293235193',
visionproBadgeExclamationmark='rbxassetid://78812515285709',airpodproRight=
'rbxassetid://74684142613428',cursorarrowClick2='rbxassetid://131954684779465',
number6AltCircle='rbxassetid://132108212531324',letterESquare=
'rbxassetid://130805805047117',letterSSquare='rbxassetid://137609708051079',
number4AltCircle='rbxassetid://120280636483094',
airpodsproChargingcaseWirelessRadiowavesLeftAndRight=
'rbxassetid://137171665646568',airpodsproChargingcaseWireless=
'rbxassetid://111093760650540',docOnDoc='rbxassetid://74221368188325',
ipodtouchSlash='rbxassetid://71827533045992',number7Circle=
'rbxassetid://91048231428377',cloudRainbowHalf='rbxassetid://80949891966526',
returnGlyph='rbxassetid://125022377592914',number38Circle=
'rbxassetid://140658654909786',number35Circle='rbxassetid://128638587888904',
arrowTriangleBranch='rbxassetid://106848864851086',number41Circle=
'rbxassetid://81051156411638',number47Square='rbxassetid://130355817586206',
beatsStudiobudsplus='rbxassetid://71975225998517',noteText=
'rbxassetid://128260562606530',number10Lane='rbxassetid://81314525759681',
letterRJoystickTiltLeft='rbxassetid://105136001525486',numeric4h=
'rbxassetid://111499849309749',number41Square='rbxassetid://108180042091768',
letterXCircle='rbxassetid://99920343804208',number36Circle=
'rbxassetid://124580301053177',number4AltSquare='rbxassetid://122930727055244',
number06Circle='rbxassetid://137452105712472',number2Square=
'rbxassetid://135434406067611',letterOSquare='rbxassetid://100189064332452',
airplayvideoBadgeExclamationmark='rbxassetid://118815409444339',
noteTextBadgePlus='rbxassetid://103069459660634',docAppend=
'rbxassetid://92502742629439',arrowCounterclockwiseIcloud=
'rbxassetid://76284977441664',steeringwheelAndLock='rbxassetid://74743575328404'
,figureSeatedSideAirbagOff='rbxassetid://104984860910066',number40Circle=
'rbxassetid://76284811463961',ipadAndIphone='rbxassetid://133274203016358',
figureSeatedSideAirDistributionUpperAngledAndMiddle=
'rbxassetid://110778248234270',number05Circle='rbxassetid://130314809611820',
creditcardAnd123='rbxassetid://87625010126625',letterJSquareOnSquare=
'rbxassetid://136068707644750',letterNSquare='rbxassetid://134128630622949',
gobackward45='rbxassetid://93434813605748',docPlaintext=
'rbxassetid://116104151169652',number44Square='rbxassetid://74196270639498',
number18Square='rbxassetid://80311469813477',airplayvideo=
'rbxassetid://84461284428154',arrowUpDocOnClipboard=
'rbxassetid://99871373237994',footballCircle='rbxassetid://82719329435185',
iphoneAndArrowForward='rbxassetid://117210262149661',number49Circle=
'rbxassetid://85012410965666',figureSeatedSideAirDistributionMiddleAndLower=
'rbxassetid://118216495126795',beatsStudiobudRight='rbxassetid://92812822483138'
,number2Brakesignal='rbxassetid://130026522834900',number45Square=
'rbxassetid://112633410131411',ipodtouchLandscape='rbxassetid://84719307699005',
gobackward90='rbxassetid://108207845332277',goforward=
'rbxassetid://131297950123949',number1Brakesignal='rbxassetid://79144309389823',
arrowTriangleSwap='rbxassetid://127274138579155',letterFCursiveCircle=
'rbxassetid://88414197139904',number17Square='rbxassetid://73499070638265',
visionproAndArrowForward='rbxassetid://108237299822657',goforward45=
'rbxassetid://90174945269280',abc='rbxassetid://100440409146443',goforward5=
'rbxassetid://92986838147448',number1Magnifyingglass=
'rbxassetid://111096049925424',letterHSquareOnSquare=
'rbxassetid://138633232751099',cursorarrow='rbxassetid://76353347413637',
letterKSquare='rbxassetid://132054295015575',arrowCirclepath=
'rbxassetid://82375876675933',number5Lane='rbxassetid://80082125386311',
homepodAndHomepodmini='rbxassetid://128138460912737',dollarsignArrowCirclepath=
'rbxassetid://111141236452341',carTopFrontleftArrowtriangle=
'rbxassetid://74249929591233',musicMic='rbxassetid://126510932173223',
number13Circle='rbxassetid://117905874629917',letterQSquare=
'rbxassetid://138412895934294',letterHSquare='rbxassetid://119444843840302',
homepodminiAndAppletv='rbxassetid://96899217298462',number14Square=
'rbxassetid://87120163705904',number5Circle='rbxassetid://84249571663594',
number6Square='rbxassetid://136827465810666',wandAndStars=
'rbxassetid://95449961416091',airplayaudio='rbxassetid://132916942955864',
number07Circle='rbxassetid://102549222409891',
figureSeatedSideAirDistributionUpperAngledAndLowerAngled=
'rbxassetid://128830311959827',letterUSquare='rbxassetid://76708256054317',
number06Square='rbxassetid://103212010162769',number42Circle=
'rbxassetid://105367690120970',ipodtouch='rbxassetid://119129980411950',
docViewfinder='rbxassetid://137034209283570',number2Lane=
'rbxassetid://76895941784764',number1Lane='rbxassetid://125260612972564',
visionproCircle='rbxassetid://70783957641063',number33Square=
'rbxassetid://91723950606376',docTextBelowEcg='rbxassetid://111731877748125',
number32Circle='rbxassetid://102701322952852',letterGCircle=
'rbxassetid://101617051624970',number00Circle='rbxassetid://131779637005640',
number43Square='rbxassetid://114456062917463',number22Circle=
'rbxassetid://79147656530403',number42Square='rbxassetid://121398683227303',
numeric4l='rbxassetid://132669284716581',number34Square=
'rbxassetid://98356773601222',letterFCursive='rbxassetid://129129749457489',
letterICircle='rbxassetid://100265252155056',letterJCircle=
'rbxassetid://127681856901613',letterK='rbxassetid://93236576104824',number6Lane
='rbxassetid://120411900425268',number35Square='rbxassetid://114874576219731',
number28Square='rbxassetid://101516015146640',letterLJoystickTiltLeft=
'rbxassetid://88760036840814',docBadgePlus='rbxassetid://124537298304533',
iphoneAndArrowLeftAndArrowRight='rbxassetid://104339904723080',letterHCircle=
'rbxassetid://104755888415516',number45Circle='rbxassetid://131982584078162',
letterBCircle='rbxassetid://81796888108964',number48Circle=
'rbxassetid://100680920554029',
figureSeatedSideWindshieldFrontAndHeatWavesAirDistributionUpperAndLower=
'rbxassetid://118828536958349',micSquare='rbxassetid://113336595252301',
letterOCircle='rbxassetid://102216372092166',letterRJoystickTiltDown=
'rbxassetid://107724362619264',letterRJoystickTiltRight=
'rbxassetid://110960582214040',arrowUpAndDownRighttriangleUpRighttriangleDown=
'rbxassetid://89990169721189',number10Circle='rbxassetid://96193951740646',
number30Circle='rbxassetid://93539842298014',letterUCircle=
'rbxassetid://100569535331718',figureRower='rbxassetid://101478600974292',
number16Square='rbxassetid://127190639879155',beatsFitPro=
'rbxassetid://122083986744951',number01Circle='rbxassetid://138017951861028',
arrowTriangle2Circlepath='rbxassetid://97876840821545',football=
'rbxassetid://99824306110508',letterVSquare='rbxassetid://129063617167906',
number17Circle='rbxassetid://104117985546346',number03Circle=
'rbxassetid://105983470184277',arrowTriangleTurnUpRightCircle=
'rbxassetid://81052567291587',number11Circle='rbxassetid://121287509877595',
arrowTriangleTurnUpRightDiamond='rbxassetid://88682763036554',docBadgeGearshape=
'rbxassetid://75017579898518',number13Square='rbxassetid://117293639837435',
number7Square='rbxassetid://128474224252993',goforward15=
'rbxassetid://91999877907911',number03Square='rbxassetid://132291497347326',
cursorarrowAndSquareOnSquareDashed='rbxassetid://123035396911468',number8Circle=
'rbxassetid://90175124244732',lockDoc='rbxassetid://122591748345523',
letterRCircle='rbxassetid://71266917331154',ipodshuffleGen4=
'rbxassetid://73762011203526',
figureSeatedSideWindshieldFrontAndHeatWavesAirDistributionUpper=
'rbxassetid://112780319350991',letterJSquare='rbxassetid://70614371401130',
gobackward10='rbxassetid://90559603880621',letterTCircle=
'rbxassetid://75623106527183',sharedWithYouSlash='rbxassetid://110559749416219',
airplayaudioBadgeExclamationmark='rbxassetid://136447514823399',letterQCircle=
'rbxassetid://121158947718797',number04Square='rbxassetid://104100419385488',
arrowRectanglepath='rbxassetid://122042553990804',macbookAndVisionpro=
'rbxassetid://127278593229556',docBadgeEllipsis='rbxassetid://131787139108704',
airpodsmax='rbxassetid://102938893282204',person2Gobackward=
'rbxassetid://77336790373231',letterLJoystickPressDown=
'rbxassetid://101991839887936',figureSeatedSideAirDistributionMiddle=
'rbxassetid://92433515109818',numeric4kTv='rbxassetid://138479967752101',
letterPCircle='rbxassetid://133454893649760',letterCCircle=
'rbxassetid://102283325788464',leafArrowTriangleCirclepath=
'rbxassetid://84475508392251',docText='rbxassetid://106048817459549',
gobackwardMinus='rbxassetid://92431716781150',hifispeakerAndHomepodmini=
'rbxassetid://104285147267424',docTextMagnifyingglass=
'rbxassetid://86983114420254',figureSkating='rbxassetid://131519789461691',
airpodspro='rbxassetid://116114855183538',
figureSeatedSideWindshieldFrontAndHeatWaves='rbxassetid://87077410641628',
number39Square='rbxassetid://110607591007885',letterRJoystickTiltUp=
'rbxassetid://71286967299311',letterSCircle='rbxassetid://83393144046860',
number02Square='rbxassetid://99179290111993',number15Square=
'rbxassetid://78052584911395',number9AltSquare='rbxassetid://127573342279334',
cursorarrowMotionlines='rbxassetid://117420120397428',letterPSquare=
'rbxassetid://121771081041226',number11Lane='rbxassetid://103036455180106',
number123Rectangle='rbxassetid://115605993616370',number12Circle=
'rbxassetid://95488831711839',number09Square='rbxassetid://87408221362281',
letterRSquare='rbxassetid://135777530848446',number34Circle=
'rbxassetid://76108893775703',number18Circle='rbxassetid://102380885337705',
number38Square='rbxassetid://87276444787352',lineDiagonalArrow=
'rbxassetid://92568642919457',number9Circle='rbxassetid://122405410433743',
number19Circle='rbxassetid://134713999920408',visionpro=
'rbxassetid://76932342920990',number46Circle='rbxassetid://98121889785827',
letterECircle='rbxassetid://110939040840018',
letterLButtonRoundedbottomHorizontal='rbxassetid://86796430696473',
figureDressLineVerticalFigure='rbxassetid://100940012571116',
arrowTriangle2CirclepathIcloud='rbxassetid://89703160887614',number1Square=
'rbxassetid://108186243137995',number21Square='rbxassetid://100575245011241',
number33Circle='rbxassetid://76331950936098',personAndArrowLeftAndArrowRight=
'rbxassetid://128164370218812',number12Square='rbxassetid://134346701257530',
docOnClipboard='rbxassetid://94277113748864',number48Square=
'rbxassetid://137504486853690',number25Circle='rbxassetid://103932230280729',
arcadeStickAndArrowLeftAndArrowRight='rbxassetid://113538637882184',docCircle=
'rbxassetid://114937888569701',gearshapeArrowTriangle2Circlepath=
'rbxassetid://84641602565527',number25Square='rbxassetid://85964236323828',
numeric4lCircle='rbxassetid://81144013373119',letterLJoystickTiltRight=
'rbxassetid://123427118118507',number29Square='rbxassetid://126060868481076',
micSlashCircle='rbxassetid://124310703535746',
gaugeOpenWithLinesNeedle33percentAndArrowtriangleFrom0percentTo50percent=
'rbxassetid://81580637007677',carTopRearleftArrowtriangle=
'rbxassetid://132477167189845',number27Square='rbxassetid://78907626263409',
number32Square='rbxassetid://97548734649989',number24Square=
'rbxassetid://102948515141281',number36Square='rbxassetid://128676757513519',
letterLJoystickTiltUp='rbxassetid://108699411245416',letterASquare=
'rbxassetid://123876841192249',number43Circle='rbxassetid://116742062830209',
number3Square='rbxassetid://100151903595968',wandAndStarsInverse=
'rbxassetid://108387310992441',beatsPowerbeatsproLeft=
'rbxassetid://132454103156247',gobackward5='rbxassetid://97130008586442',
figureSeatedSideAutomatic='rbxassetid://136215915659039',number14Circle=
'rbxassetid://108407748009645',number30Square='rbxassetid://92360148508407',
homepodmini2='rbxassetid://138072319954848',number7Lane=
'rbxassetid://89152600086974',number8Square='rbxassetid://103037928409169',
number9Lane='rbxassetid://118878335756561',
arrowLeftAndRightRighttriangleLeftRighttriangleRight=
'rbxassetid://103119872066027',letterLCircle='rbxassetid://115151845775938',
number40Square='rbxassetid://108964733911464',letterLJoystick=
'rbxassetid://126954698608958',letterMSquare='rbxassetid://91788346353476',
airplayvideoCircle='rbxassetid://76641158810884',goforward90=
'rbxassetid://81147746129102',micAndSignalMeter='rbxassetid://98371685797325',
letterVCircle='rbxassetid://83920398217229',flagCheckeredCircle=
'rbxassetid://123648433572775',cursorarrowSlashSquare=
'rbxassetid://95513304668296',number8Lane='rbxassetid://135579479621822',
carTopFrontrightArrowtriangle='rbxassetid://109439050535736',
letterRSquareOnSquare='rbxassetid://123053148394613',gobackward75=
'rbxassetid://140735752535943',letterFSquare='rbxassetid://80222891036830',
visionproSlashCircle='rbxassetid://116051094478142',number3Circle=
'rbxassetid://82210102360256',number20Circle='rbxassetid://119790353650997',
number49Square='rbxassetid://112216977302610',
figureSeatedSideAirDistributionLower='rbxassetid://120257665940386',
dotsAndLineVerticalAndCursorarrowRectangle='rbxassetid://82575719013218',
beatsPowerbeatspro='rbxassetid://91388282815385',letterXSquare=
'rbxassetid://123476212380657',micBadgeXmark='rbxassetid://90776924897027',
letterYCircle='rbxassetid://120543405393753',musicMicCircle=
'rbxassetid://121995856123205',number26Circle='rbxassetid://110095157711014',
arrowRightDocOnClipboard='rbxassetid://84583825314018',number5Square=
'rbxassetid://92555254652509',number31Circle='rbxassetid://117202855882818',
letterZSquare='rbxassetid://139916717288653',arrowUpDoc=
'rbxassetid://130298316589925',gobackward='rbxassetid://91320889824011',
carTopRearrightArrowtriangle='rbxassetid://121164497326761',docTextImage=
'rbxassetid://122105011236494',beatsFitProChargingcase=
'rbxassetid://120936645934230',rectangleCheckered='rbxassetid://93153326915076',
visionproSlash='rbxassetid://118082278749251',macwindowAndCursorarrow=
'rbxassetid://90741041437063',sharedWithYouCircle='rbxassetid://105105656683870'
,caseGylph='rbxassetid://72684442825623',goforward10=
'rbxassetid://136515914036455',gymBag='rbxassetid://91936141424860',
docBadgeArrowUp='rbxassetid://80297421254328',homepodmini=
'rbxassetid://93368409427047',exclamationmarkArrowTriangle2Circlepath=
'rbxassetid://121048701816249',number27Circle='rbxassetid://80413679989648',
airplayaudioCircle='rbxassetid://88830131647623',number39Circle=
'rbxassetid://127160538226349',beatsStudiobudLeft='rbxassetid://123610938062441'
,gobackward15='rbxassetid://120316938540272',number15Circle=
'rbxassetid://101007633009470',waterWavesAndArrowDown=
'rbxassetid://108671761307420',number50Circle='rbxassetid://76596482604277',
contextualmenuAndCursorarrow='rbxassetid://123228274012415',numeric2hCircle=
'rbxassetid://70624293530403',number21Circle='rbxassetid://115011080119497',
beatsStudiobudsplusLeft='rbxassetid://111603577383991',arrowClockwiseHeart=
'rbxassetid://89932974116238',gobackward60='rbxassetid://91215922364999',
number00Square='rbxassetid://81236373352019',sharedWithYou=
'rbxassetid://99587894742741',envelopeArrowTriangleBranch=
'rbxassetid://124377309512740',docRichtext='rbxassetid://94749196343692',
number47Circle='rbxassetid://110683991524351',letterWSquare=
'rbxassetid://83322547030564',arrowTriangle2CirclepathCamera=
'rbxassetid://110761440206205',beatsPowerbeatsproRight=
'rbxassetid://121014913156174',figureSeatedSideAirDistributionUpper=
'rbxassetid://80332126775420',letterTSquare='rbxassetid://116018632541398',
number37Circle='rbxassetid://105355724679695',number08Circle=
'rbxassetid://133051884093292',number4Lane='rbxassetid://80369915269072',
micBadgePlus='rbxassetid://124905697228070',micSlash=
'rbxassetid://77413617305085',ipadAndIphoneSlash='rbxassetid://98585690423131',
number6Circle='rbxassetid://139508584419391',sliderHorizontal2Gobackward=
'rbxassetid://82873088218553',number2Circle='rbxassetid://115644023941063',
letterGSquare='rbxassetid://101108516134256',circleBottomrighthalfCheckered=
'rbxassetid://122575558098594',docBadgeClock='rbxassetid://106217049362270',
letterDSquare='rbxassetid://71745998559849',filemenuAndCursorarrow=
'rbxassetid://103877132330792',clockArrow2Circlepath=
'rbxassetid://120448260426448',arrowClockwiseIcloud=
'rbxassetid://119564317606359',number24Circle='rbxassetid://77688026724586',
figureSeatedSideWindshieldFrontAndHeatWavesAirDistributionUpperAndMiddle=
'rbxassetid://117604588619581',
sliderHorizontal2RectangleAndArrowTriangle2Circlepath=
'rbxassetid://114334030505190',
figureSeatedSideAirDistributionMiddleAndLowerAngled=
'rbxassetid://132877218661072',goforward75='rbxassetid://71594405050565',
arrowTrianglePull='rbxassetid://92844954623247',number20Square=
'rbxassetid://73427706736721',clockArrowCirclepath='rbxassetid://83500708276723'
,homekit='rbxassetid://129816863247706',number04Circle=
'rbxassetid://116987436384055',cursorarrowSquare='rbxassetid://131316405162767',
letterWCircle='rbxassetid://107802977183125',repeatGlyph=
'rbxassetid://94715683784128',number4Circle='rbxassetid://113950512097158',
goforward60='rbxassetid://116153177521739',mic='rbxassetid://70559142840213',
figureSeatedSideAirbagOn2='rbxassetid://133224719349931',number16Circle=
'rbxassetid://107648080370730',letterFCircle='rbxassetid://75292206915955',
dotCircleAndCursorarrow='rbxassetid://115122421262820',goforward30=
'rbxassetid://77322694446691'}end function a.g()local c=a.cache.g if not c then
c={c=b()}a.cache.g=c end return c.c end end do local b=function()local b,c=a.e()
,a.d()local d,e,f,g=b.Workspace,b.Lighting,c.Create,function(d,e,f,g,h)return(d-
e)*(h-g)/(f-e)+g end local h,i=function(h,i)local j=d.CurrentCamera:
ScreenPointToRay(h.X,h.Y)return j.Origin+j.Direction*i end,function()local h=d.
CurrentCamera.ViewportSize.Y return g(h,0,2560,8,56)end local j=function(j)local
k,l,m,n,o=Instance.new('Folder',d.CurrentCamera),{},{topLeft=Vector2.new(),
topRight=Vector2.new(),bottomRight=Vector2.new()},(f'Part'{Color=Color3.new(0,0,
0),Material=Enum.Material.Glass,Size=Vector3.new(1,1,0),Anchored=true,CanTouch=
false,CanCollide=false,CanQuery=false,Locked=true,CastShadow=false,Transparency=
0.98,f'SpecialMesh'{MeshType=Enum.MeshType.Brick,Offset=Vector3.new(0,0,-1E-6)}}
),f'DepthOfFieldEffect'{FarIntensity=0,NearIntensity=1,InFocusRadius=0.1,Parent=
e}local p=n:FindFirstChildWhichIsA'SpecialMesh'j=j or 0.001 local q,r=function(q
,r)m.topLeft=r m.topRight=r+Vector2.new(q.X,0)m.bottomRight=r+q end,function()
local q=d.CurrentCamera local r,s,t,u=(q and{(q.CFrame)}or{(CFrame.identity)})[1
],m.topLeft,m.topRight,m.bottomRight local v,w,x=h(s,j),h(t,j),h(u,j)local y,z=(
w-v).Magnitude,(w-x).Magnitude n.CFrame=CFrame.fromMatrix((v+x)/2,r.XVector,r.
YVector,r.ZVector)if p then p.Scale=Vector3.new(y,z,0)end end local s,t=function
(s)local t=i()local u,v=s.AbsoluteSize-Vector2.new(t,t),s.AbsolutePosition+
Vector2.new(t/2,t/2)q(u,v)task.spawn(r)end,function()local s=d.CurrentCamera if
not s then return end l[#l+1]=s:GetPropertyChangedSignal'CFrame':Connect(r)l[#l+
1]=s:GetPropertyChangedSignal'ViewportSize':Connect(r)l[#l+1]=s:
GetPropertyChangedSignal'FieldOfView':Connect(r)task.spawn(r)end n.Parent=k n.
Destroying:Connect(function()for u,v in l do pcall(function()o:Destroy()k:
Destroy()v:Disconnect()end)end end)t()return s,n end return function(k,l)local m
,n,o={},j(l)m.SetVisibility=function(p)o.Transparency=p and 0.98 or 1 end k:
GetPropertyChangedSignal'AbsolutePosition':Connect(function()n(k)end)k:
GetPropertyChangedSignal'AbsoluteSize':Connect(function()n(k)end)k:
GetPropertyChangedSignal'Visible':Connect(function()m.SetVisibility(k.Visible)
end)k.Destroying:Connect(function()o:Destroy()end)m.Model=o n(k)return m end end
function a.h()local c=a.cache.h if not c then c={c=b()}a.cache.h=c end return c.
c end end do local b=function()return{UIBlur=a.h()}end function a.i()local c=a.
cache.i if not c then c={c=b()}a.cache.i=c end return c.c end end do local b=
function()return function(b)local c,d=a.d(),a.e()local e,f,g=c.Create,d.
TweenService,{}g.Body=(e'Frame'{Name='TextField',AutomaticSize=Enum.
AutomaticSize.XY,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Size=UDim2.fromOffset(150,23),SizeConstraint=Enum.SizeConstraint.RelativeXX,e
'UICorner'{Name='UICorner',CornerRadius=UDim.new(0,5)},e'UIStroke'{Name=
'UIStroke',ApplyStrokeMode=Enum.ApplyStrokeMode.Border,Thickness=1,__dynamicKeys
={Color=b.Theme.Controls.ViewBorder[1],Transparency=b.Theme.Controls.ViewBorder[
2]}},e'TextBox'{Name='Field',AutomaticSize=Enum.AutomaticSize.XY,Size=UDim2.
fromOffset(138,0),BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
CursorPosition=-1,TextXAlignment=Enum.TextXAlignment.Left,FontFace=Font.new
'rbxassetid://12187365364',PlaceholderText='',Text='',TextSize=15,__dynamicKeys=
{PlaceholderColor3=b.Theme.Text.Tertiary[1],TextTransparency=b.Theme.Text.
Tertiary[2]},__contextKeys={_general=function()local h=g.Field if not h then
return end if h.Text==''then h.PlaceholderColor3=b.Theme.Text.Tertiary[1].Value
h.TextTransparency=b.Theme.Text.Tertiary[2].Value else h.TextColor3=b.Theme.Text
.Primary[1].Value h.TextTransparency=b.Theme.Text.Primary[2].Value end end,
TextTransparency=function()if not g.Field then return 1 end return g.Field.Text
==''and b.Theme.Text.Tertiary[2].Value or b.Theme.Text.Primary[2].Value end},e
'UIPadding'{Name='UIPadding',PaddingTop=UDim.new(0,4),PaddingBottom=UDim.new(0,4
)}},e'UIPadding'{Name='UIPadding',PaddingLeft=UDim.new(0,6),PaddingRight=UDim.
new(0,6)},e'UIListLayout'{Name='UIListLayout',SortOrder=Enum.SortOrder.
LayoutOrder,HorizontalAlignment=Enum.HorizontalAlignment.Right,VerticalAlignment
=Enum.VerticalAlignment.Center,FillDirection=Enum.FillDirection.Horizontal,
Padding=UDim.new(0,4)}})g.Stroke=(g.Body:FindFirstChild'UIStroke')g.Padding=(g.
Body:FindFirstChild'UIPadding')g.Corner=(g.Body:FindFirstChild'UICorner')g.
Layout=(g.Body:FindFirstChild'UIListLayout')g.Field=(g.Body:FindFirstChild
'Field')g.FieldPadding=(g.Field:FindFirstChild'UIPadding')local h,i,j=TweenInfo.
new(0.13,Enum.EasingStyle.Sine,Enum.EasingDirection.Out),(TweenInfo.new(0.2,Enum
.EasingStyle.Sine,Enum.EasingDirection.Out))g.Field.Focused:Connect(function()if
j then j:Cancel()end j=f:Create(g.Stroke,h,{Color=b.Theme.Controls.
SelectionStroke[1].Value,Transparency=b.Theme.Controls.SelectionStroke[2].Value,
Thickness=3})j:Play()end)g.Field.FocusLost:Connect(function()if j then j:Cancel(
)end j=f:Create(g.Stroke,i,{Color=b.Theme.Controls.ViewBorder[1].Value,
Transparency=b.Theme.Controls.ViewBorder[2].Value,Thickness=1})j:Play()end)g.
Field:GetPropertyChangedSignal'Text':Connect(function()if g.Field.Text==''then g
.Field.PlaceholderColor3=b.Theme.Text.Tertiary[1].Value g.Field.TextTransparency
=b.Theme.Text.Tertiary[2].Value else g.Field.TextColor3=b.Theme.Text.Primary[1].
Value g.Field.TextTransparency=b.Theme.Text.Primary[2].Value end end)return g
end end function a.j()local c=a.cache.j if not c then c={c=b()}a.cache.j=c end
return c.c end end do local b=function()return function(b)local c,d=a.d(),a.j()(
b)local e,f,g,h,i,j=c.Create,{},0,{},function(e)local f,g=e.AbsolutePosition,e.
AbsoluteSize return{Left=f.X,Right=f.X+g.X,Top=f.Y,Bottom=f.Y+g.Y}end h.Body=(e
'Frame'{Name='Toolbar',BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Size
=UDim2.new(1,0,0,52),__dynamicKeys={BackgroundColor3=b.Theme.Controls.Titlebar[1
],BackgroundTransparency=b.Theme.Controls.Titlebar[2]},e'UICorner'{Name=
'UICorner',CornerRadius=UDim.new(0,10)},e'Frame'{Name='Shadow',AnchorPoint=
Vector2.new(1,0),BackgroundColor3=Color3.fromRGB(234,234,234),
BackgroundTransparency=0.75,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0
,Position=UDim2.fromScale(1,1),Size=UDim2.new(1,0,0,2),ZIndex=0,__dynamicKeys={
BackgroundColor3=b.Theme.Controls.TitlebarShadow.Background[1],
BackgroundTransparency=b.Theme.Controls.TitlebarShadow.Background[2]},e
'UIGradient'{Name='UIGradient',Color=ColorSequence.new{ColorSequenceKeypoint.
new(0,Color3.fromRGB(255,255,255)),ColorSequenceKeypoint.new(1,Color3.fromRGB(0,
0,0))},Rotation=-90,Transparency=NumberSequence.new{NumberSequenceKeypoint.new(0
,0.35),NumberSequenceKeypoint.new(1,0.35)},__dynamicKeys={Color=b.Theme.Controls
.TitlebarShadow.Color,Transparency=b.Theme.Controls.TitlebarShadow.Transparency}
}},e'Frame'{Name='Content',BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Size=UDim2.fromScale(1,1),e'Frame'{Name='Leading',BackgroundColor3=Color3.
fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0)
,BorderSizePixel=0,Size=UDim2.fromScale(0,1),e'UIListLayout'{Name='UIListLayout'
,FillDirection=Enum.FillDirection.Horizontal,Padding=UDim.new(0,8),SortOrder=
Enum.SortOrder.LayoutOrder,VerticalAlignment=Enum.VerticalAlignment.Center},e
'Frame'{Name='TitleSubtitle',AutomaticSize=Enum.AutomaticSize.XY,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,LayoutOrder=2,e
'UIListLayout'{Name='UIListLayout',SortOrder=Enum.SortOrder.LayoutOrder,
VerticalAlignment=Enum.VerticalAlignment.Center},e'TextLabel'{Name='Subtitle',
AutomaticSize=Enum.AutomaticSize.X,BackgroundTransparency=1,BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,FontFace=Font.new('rbxassetid://12187365364',
Enum.FontWeight.Medium,Enum.FontStyle.Normal),LayoutOrder=1,RichText=true,Size=
UDim2.fromOffset(0,14),Text='Subtitle',TextColor3=Color3.fromRGB(0,0,0),TextSize
=12,Visible=false,__dynamicKeys={TextColor3=b.Theme.Text.Secondary[1],
TextTransparency=b.Theme.Text.Secondary[2]}},e'TextLabel'{Name='Title',
AutomaticSize=Enum.AutomaticSize.X,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
FontFace=Font.new('rbxassetid://12187365364',Enum.FontWeight.SemiBold,Enum.
FontStyle.Normal),LineHeight=0,RichText=true,Size=UDim2.fromOffset(0,20),Text=
'Title',TextSize=16,__dynamicKeys={TextColor3=b.Theme.Text.Primary[1],
TextTransparency=b.Theme.Text.Primary[2]}}},e'UIPadding'{Name='UIPadding',
PaddingLeft=UDim.new(0,12),PaddingRight=UDim.new(0,12)},e'Frame'{Name=
'NavigationButtons',AutomaticSize=Enum.AutomaticSize.XY,BackgroundColor3=Color3.
fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0)
,BorderSizePixel=0,LayoutOrder=1,e'UIListLayout'{Name='UIListLayout',
FillDirection=Enum.FillDirection.Horizontal,Padding=UDim.new(0,1),SortOrder=Enum
.SortOrder.LayoutOrder,VerticalAlignment=Enum.VerticalAlignment.Center},e
'TextButton'{Name='Back',BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
FontFace=Font.new'rbxasset://fonts/families/SourceSansPro.json',Size=UDim2.
fromOffset(27,20),Text='',TextColor3=Color3.fromRGB(0,0,0),TextSize=14,e
'ImageLabel'{Name='Image',AnchorPoint=Vector2.new(0.5,0.5),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Image='rbxassetid://137248392050045',ImageColor3=Color3.fromRGB(0,0,0),Position=
UDim2.fromScale(0.5,0.5),Size=UDim2.fromOffset(9,15),__dynamicKeys={ImageColor3=
b.Theme.Text.Tertiary[1],ImageTransparency=b.Theme.Text.Tertiary[2]},
__contextKeys={ImageColor3=function()local k,l=g>1,j.Theme.Text return(k and l.
Secondary[1].Value)or l.Tertiary[1].Value end,ImageTransparency=function()local
k,l=g>1,j.Theme.Text return(k and l.Secondary[2].Value)or l.Tertiary[2].Value
end}}},e'TextButton'{Name='Forward',BackgroundColor3=Color3.fromRGB(255,255,255)
,BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
FontFace=Font.new'rbxasset://fonts/families/SourceSansPro.json',LayoutOrder=1,
Size=UDim2.fromOffset(27,20),Text='',TextColor3=Color3.fromRGB(0,0,0),TextSize=
14,e'ImageLabel'{Name='Image',AnchorPoint=Vector2.new(0.5,0.5),BackgroundColor3=
Color3.fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,Image='rbxassetid://95285878898359',Position=
UDim2.fromScale(0.5,0.5),Size=UDim2.fromOffset(9,15),__dynamicKeys={ImageColor3=
b.Theme.Text.Tertiary[1],ImageTransparency=b.Theme.Text.Tertiary[2]},
__contextKeys={ImageColor3=function()local k,l=g<#f,j.Theme.Text return(k and l.
Secondary[1].Value)or l.Tertiary[1].Value end,ImageTransparency=function()local
k,l=g<#f,j.Theme.Text return(k and l.Secondary[2].Value)or l.Tertiary[2].Value
end}}}},e'Frame'{Name='SidebarButton',AutomaticSize=Enum.AutomaticSize.XY,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Size=UDim2.fromOffset(38,34
),e'ImageButton'{Name='Image',AnchorPoint=Vector2.new(0.5,0.5),AutoButtonColor=
false,BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://74380920233260',Position=UDim2.fromScale(0.5,0.5),Size=UDim2.
fromOffset(20,16),__dynamicKeys={ImageColor3=b.Theme.Text.Secondary[1],
ImageTransparency=b.Theme.Text.Secondary[2]}}}},e'UIListLayout'{Name=
'UIListLayout',FillDirection=Enum.FillDirection.Horizontal,HorizontalFlex=Enum.
UIFlexAlignment.Fill,SortOrder=Enum.SortOrder.LayoutOrder},e'Frame'{Name=
'Trailing',BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1
,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Size=UDim2.fromScale(0,1),
e'UIPadding'{Name='UIPadding',PaddingLeft=UDim.new(0,12),PaddingRight=UDim.new(0
,12)},e'UIListLayout'{Name='UIListLayout',HorizontalAlignment=Enum.
HorizontalAlignment.Right,SortOrder=Enum.SortOrder.LayoutOrder,VerticalAlignment
=Enum.VerticalAlignment.Center}}},e'Frame'{Name='CornerClipRight',AnchorPoint=
Vector2.new(1,1),BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Position=
UDim2.fromScale(1,1),Size=UDim2.new(0,10,0.5,0),ZIndex=-1,__dynamicKeys={
BackgroundColor3=b.Theme.Controls.Titlebar[1]}},e'Frame'{Name='CornerClipLeft',
AnchorPoint=Vector2.new(0,1),BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=
0,Position=UDim2.fromScale(0,1),Size=UDim2.new(0,10,1,0),ZIndex=-1,__dynamicKeys
={BackgroundColor3=b.Theme.Controls.Titlebar[1]}}})h.CornerClipLeft=(h.Body:
FindFirstChild'CornerClipLeft')h.CornerClipRight=(h.Body:FindFirstChild
'CornerClipRight')h.Content=(h.Body:FindFirstChild'Content')h.Leading=(h.Content
:FindFirstChild'Leading')h.Trailing=(h.Content:FindFirstChild'Trailing')h.
NavigationButtons=(h.Leading:FindFirstChild'NavigationButtons')h.Back=(h.
NavigationButtons:FindFirstChild'Back')h.Forward=(h.NavigationButtons:
FindFirstChild'Forward')h.SidebarButton=(h.Leading:FindFirstChild'SidebarButton'
:FindFirstChild'Image')h.TitleStack=(h.Leading:FindFirstChild'TitleSubtitle')h.
Title=(h.TitleStack:FindFirstChild'Title')h.Subtitle=(h.TitleStack:
FindFirstChild'Subtitle')h.SearchField=d h.SearchField.Body.Size=UDim2.
fromOffset(126,28)h.SearchField.Field.Size=UDim2.fromOffset(90,0)h.SearchField.
Layout.Padding=UDim.new(0,8)h.SearchField.Field.LayoutOrder=1 h.SearchField.
Field.PlaceholderText='Search'h.SearchField.Padding.PaddingLeft=UDim.new(0,7)h.
SearchField.Padding.PaddingRight=UDim.new(0,7)h.SearchField.Corner=UDim.new(0,6)
e'ImageLabel'{Name='ImageLabel',BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Image='rbxassetid://120817720539967',Size=UDim2.fromOffset(13,14),Parent=h.
SearchField.Body.__instance,__dynamicKeys={ImageColor3=b.Theme.Text.
PrimaryAccent[1],ImageTransparency=b.Theme.Text.PrimaryAccent[2]}}h.SearchField.
Body.Parent=h.Trailing local k=function()if not j then return end local k,l,m=h.
Back:FindFirstChild'Image',h.Forward:FindFirstChild'Image',j.Theme.Text if k
then local n=g>1 k.ImageColor3=(n and m.Secondary[1].Value)or m.Tertiary[1].
Value k.ImageTransparency=(n and m.Secondary[2].Value)or m.Tertiary[2].Value h.
Back.Interactable=n end if l then local n=g<#f l.ImageColor3=(n and m.Secondary[
1].Value)or m.Tertiary[1].Value l.ImageTransparency=(n and m.Secondary[2].Value)
or m.Tertiary[2].Value h.Forward.Interactable=n end end local l,m=setmetatable({
},{__newindex=function(l,m,n)rawset(f,m,n)k()end,__index=function(l,m)return f[m
]end,__len=function()return#f end}),function(l)if j and j.Tabs then for m,n in
pairs(j.Tabs)do if n.__container==l then n.Selected=true else n.Selected=false
end end end end local n,o=function()h.Back.MouseButton1Click:Connect(function()
if g>1 then g-=1 m(f[g])k()end end)h.Forward.MouseButton1Click:Connect(function(
)if g<#f then g+=1 m(f[g])k()end end)end,function()for n,o in pairs(h.Trailing:
GetChildren())do if not o:IsA'GuiObject'then continue end local p,q=i(o),true
for r,s in pairs(h.Leading:GetChildren())do if not s:IsA'GuiObject'then continue
end local t=i(s)local u=(t.Right>=p.Left and t.Left<=p.Right)and(t.Bottom>=p.Top
and t.Top<=p.Bottom)if u then q=false break end end o.Visible=q end end o()h.
Body:GetPropertyChangedSignal'AbsoluteSize':Connect(o)return h,function(p)j=p n(
)return function(q)for r=#f,g+1,-1 do f[r]=nil end g+=1 l[g]=q end end end end
function a.k()local c=a.cache.k if not c then c={c=b()}a.cache.k=c end return c.
c end end do local b=function()return function(b,c)local d=a.d()local e,f=d.
Create,{}f.Handles=(e'Folder'{Name='Handles',Parent=b,e'Frame'{Name='E',
AnchorPoint=Vector2.new(1,0.5),BackgroundColor3=Color3.fromRGB(255,0,81),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Position=UDim2.fromScale(1,0.5),Size=UDim2.new(0,6,1,0),ZIndex=8,e
'UIDragDetector'{Name='UIDragDetector',ActivatedCursorIcon=
'rbxassetid://122180269146343',CursorIcon='rbxassetid://122180269146343',
ResponseStyle=Enum.UIDragDetectorResponseStyle.CustomOffset,
SelectionModeDragSpeed=UDim2.new(),SelectionModeRotateSpeed=0}},e'Frame'{Name=
'N',AnchorPoint=Vector2.new(0.5,0),BackgroundColor3=Color3.fromRGB(255,0,81),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Position=UDim2.fromScale(0.5,0),Size=UDim2.new(1,0,0,6),ZIndex=8,e
'UIDragDetector'{Name='UIDragDetector',ActivatedCursorIcon=
'rbxassetid://109321922977357',CursorIcon='rbxassetid://109321922977357',
ResponseStyle=Enum.UIDragDetectorResponseStyle.CustomOffset,
SelectionModeDragSpeed=UDim2.new(),SelectionModeRotateSpeed=0}},e'Frame'{Name=
'NE',AnchorPoint=Vector2.new(1,0),BackgroundColor3=Color3.fromRGB(255,0,81),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Position=UDim2.fromScale(1,0),Size=UDim2.fromOffset(10,10),ZIndex=9,e
'UIDragDetector'{Name='UIDragDetector',ActivatedCursorIcon=
'rbxassetid://78578587100983',CursorIcon='rbxassetid://78578587100983',
ResponseStyle=Enum.UIDragDetectorResponseStyle.CustomOffset,
SelectionModeDragSpeed=UDim2.new(),SelectionModeRotateSpeed=0}},e'Frame'{Name=
'NW',BackgroundColor3=Color3.fromRGB(255,0,81),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Size=UDim2.fromOffset(10,10
),ZIndex=9,e'UIDragDetector'{Name='UIDragDetector',ActivatedCursorIcon=
'rbxassetid://127507239458439',CursorIcon='rbxassetid://127507239458439',
ResponseStyle=Enum.UIDragDetectorResponseStyle.CustomOffset,
SelectionModeDragSpeed=UDim2.new(),SelectionModeRotateSpeed=0}},e'Frame'{Name=
'S',AnchorPoint=Vector2.new(0.5,1),BackgroundColor3=Color3.fromRGB(255,0,81),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Position=UDim2.fromScale(0.5,1),Size=UDim2.new(1,0,0,6),ZIndex=8,e
'UIDragDetector'{Name='UIDragDetector',ActivatedCursorIcon=
'rbxassetid://109321922977357',CursorIcon='rbxassetid://109321922977357',
ResponseStyle=Enum.UIDragDetectorResponseStyle.CustomOffset,
SelectionModeDragSpeed=UDim2.new(),SelectionModeRotateSpeed=0}},e'Frame'{Name=
'SE',AnchorPoint=Vector2.new(1,1),BackgroundColor3=Color3.fromRGB(255,0,81),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Position=UDim2.fromScale(1,1),Size=UDim2.fromOffset(10,10),ZIndex=9,e
'UIDragDetector'{Name='UIDragDetector',ActivatedCursorIcon=
'rbxassetid://127507239458439',CursorIcon='rbxassetid://127507239458439',
ResponseStyle=Enum.UIDragDetectorResponseStyle.CustomOffset,
SelectionModeDragSpeed=UDim2.new(),SelectionModeRotateSpeed=0}},e'Frame'{Name=
'SW',AnchorPoint=Vector2.new(0,1),BackgroundColor3=Color3.fromRGB(255,0,81),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Position=UDim2.fromScale(0,1),Size=UDim2.fromOffset(10,10),ZIndex=9,e
'UIDragDetector'{Name='UIDragDetector',ActivatedCursorIcon=
'rbxassetid://78578587100983',CursorIcon='rbxassetid://78578587100983',
ResponseStyle=Enum.UIDragDetectorResponseStyle.CustomOffset,
SelectionModeDragSpeed=UDim2.new(),SelectionModeRotateSpeed=0}},e'Frame'{Name=
'W',AnchorPoint=Vector2.new(0,0.5),BackgroundColor3=Color3.fromRGB(255,0,81),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Position=UDim2.fromScale(0,0.5),Size=UDim2.new(0,6,1,0),ZIndex=8,e
'UIDragDetector'{Name='UIDragDetector',ActivatedCursorIcon=
'rbxassetid://122180269146343',CursorIcon='rbxassetid://122180269146343',
ResponseStyle=Enum.UIDragDetectorResponseStyle.CustomOffset,
SelectionModeDragSpeed=UDim2.new(),SelectionModeRotateSpeed=0}}})c=c or Vector2.
new(0,0)f.N=f.Handles:FindFirstChild'N':FindFirstChild'UIDragDetector'f.S=f.
Handles:FindFirstChild'S':FindFirstChild'UIDragDetector'f.W=f.Handles:
FindFirstChild'W':FindFirstChild'UIDragDetector'f.E=f.Handles:FindFirstChild'E':
FindFirstChild'UIDragDetector'f.NE=f.Handles:FindFirstChild'NE':FindFirstChild
'UIDragDetector'f.SE=f.Handles:FindFirstChild'SE':FindFirstChild'UIDragDetector'
f.NW=f.Handles:FindFirstChild'NW':FindFirstChild'UIDragDetector'f.SW=f.Handles:
FindFirstChild'SW':FindFirstChild'UIDragDetector'local g,h,i={}g.__index=g local
j=function(j,k)local l=Vector2.zero if string.find(k,'N')then l+=Vector2.new(0,-
1)end if string.find(k,'S')then l+=Vector2.new(0,1)end if string.find(k,'W')then
l+=Vector2.new(-1,0)end if string.find(k,'E')then l+=Vector2.new(1,0)end local m
=Vector2.new((l.X~=0)and(j.X*l.X)or 0,(l.Y~=0)and(j.Y*l.Y)or 0)local n=i+m n=
Vector2.new(math.max(c.X,n.X),math.max(c.Y,n.Y))local o=n-i local p=Vector2.new(
l.X<0 and-o.X*0.5 or o.X*0.5,l.Y<0 and-o.Y*0.5 or o.Y*0.5)local q=h+p b.Size=
UDim2.fromOffset(n.X,n.Y)b.Position=UDim2.new(0.5,q.X,0.5,q.Y)end local k=
function(k,l)k.DragStart:Connect(function()i=b.AbsoluteSize h=Vector2.new(b.
Position.X.Offset,b.Position.Y.Offset)end)k.DragContinue:Connect(function()j(
Vector2.new(k.DragUDim2.X.Offset,k.DragUDim2.Y.Offset),l)end)end function g.
CreateClient()k(f.NW,'NW')k(f.NE,'NE')k(f.SW,'SW')k(f.SE,'SE')k(f.E,'E')k(f.W,
'W')k(f.S,'S')k(f.N,'N')end function g.SetEnabled(l)for m,n in pairs(f)do if n:
IsA'UIDragDetector'then n.Enabled=l end end end return g end end function a.l()
local c=a.cache.l if not c then c={c=b()}a.cache.l=c end return c.c end end do
local b=function()return function(b,c)local d=a.d()local e,f,g=d.Create,{},{Exit
={ImageColor3=function()return c.CanExit and b.Theme.Controls.Exit[1].Value or b
.Theme.Controls.WindowControlDisabled[1].Value end,ImageTransparency=function()
return c.CanExit and b.Theme.Controls.Exit[2].Value or b.Theme.Controls.
WindowControlDisabled[2].Value end},Minimize={ImageColor3=function()return c.
CanMinimize and b.Theme.Controls.Minimize[1].Value or b.Theme.Controls.
WindowControlDisabled[1].Value end,ImageTransparency=function()return c.
CanMinimize and b.Theme.Controls.Minimize[2].Value or b.Theme.Controls.
WindowControlDisabled[2].Value end},Zoom={ImageColor3=function()return c.CanZoom
and b.Theme.Controls.Zoom[1].Value or b.Theme.Controls.WindowControlDisabled[1].
Value end,ImageTransparency=function()return c.CanZoom and b.Theme.Controls.Zoom
[2].Value or b.Theme.Controls.WindowControlDisabled[2].Value end}}f.Body=e
'Frame'{Name='WindowControls',AnchorPoint=Vector2.new(0,0.5),BackgroundColor3=
Color3.fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,Position=UDim2.fromScale(0,0.5),Size=UDim2.
fromOffset(92,38),e'UIListLayout'{Name='UIListLayout',FillDirection=Enum.
FillDirection.Horizontal,Padding=UDim.new(0,8),SortOrder=Enum.SortOrder.
LayoutOrder,VerticalAlignment=Enum.VerticalAlignment.Center},e'UIPadding'{Name=
'UIPadding',PaddingLeft=UDim.new(0,20),PaddingRight=UDim.new(0,20)}}f.Exit=e
'ImageButton'{Name='Exit',AutoButtonColor=false,BackgroundColor3=Color3.fromRGB(
255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Image='rbxassetid://132228700346004',Size=UDim2.fromOffset(12,
12),Parent=f.Body.__instance,__dynamicKeys={ImageColor3=b.Theme.Controls.Exit[1]
,ImageTransparency=b.Theme.Controls.Exit[2]},__contextKeys=g.Exit,e'ImageLabel'{
Name='Stroke',BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Image='rbxassetid://94763505860483',Size=UDim2.fromScale(1,1),__dynamicKeys={
ImageColor3=b.Theme.Controls.WindowControlStroke[1],ImageTransparency=b.Theme.
Controls.WindowControlStroke[2]}},e'ImageLabel'{Name='Icon',AnchorPoint=Vector2.
new(0.5,0.5),BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency
=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://94781753558308',Position=UDim2.fromScale(0.5,0.5),Size=UDim2.
fromScale(1,1),Visible=false,__dynamicKeys={ImageColor3=b.Theme.Controls.
WindowControlIcon[1],ImageTransparency=b.Theme.Controls.WindowControlIcon[2]}}}f
.Minimize=e'ImageButton'{Name='Minimize',AutoButtonColor=false,BackgroundColor3=
Color3.fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,Image='rbxassetid://132228700346004',
LayoutOrder=1,Size=UDim2.fromOffset(12,12),Parent=f.Body.__instance,
__dynamicKeys={ImageColor3=b.Theme.Controls.Minimize[1],ImageTransparency=b.
Theme.Controls.Minimize[2]},__contextKeys=g.Minimize,e'ImageLabel'{Name='Stroke'
,BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://94763505860483',Size=UDim2.fromScale(1,1),__dynamicKeys={
ImageColor3=b.Theme.Controls.WindowControlStroke[1],ImageTransparency=b.Theme.
Controls.WindowControlStroke[2]}},e'ImageLabel'{Name='Icon',AnchorPoint=Vector2.
new(0.5,0.5),BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency
=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://118368309445367',Position=UDim2.fromScale(0.5,0.5),Size=UDim2.
fromScale(1,1),Visible=false,__dynamicKeys={ImageColor3=b.Theme.Controls.
WindowControlIcon[1],ImageTransparency=b.Theme.Controls.WindowControlIcon[2]}}}f
.Zoom=e'ImageButton'{Name='Zoom',AutoButtonColor=false,BackgroundColor3=Color3.
fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0)
,BorderSizePixel=0,Image='rbxassetid://132228700346004',LayoutOrder=2,Size=UDim2
.fromOffset(12,12),Parent=f.Body.__instance,__dynamicKeys={ImageColor3=b.Theme.
Controls.Zoom[1],ImageTransparency=b.Theme.Controls.Zoom[2]},__contextKeys=g.
Zoom,e'ImageLabel'{Name='Stroke',BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Image='rbxassetid://94763505860483',Size=UDim2.fromScale(1,1),__dynamicKeys={
ImageColor3=b.Theme.Controls.WindowControlStroke[1],ImageTransparency=b.Theme.
Controls.WindowControlStroke[2]}},e'ImageLabel'{Name='Icon',AnchorPoint=Vector2.
new(0.5,0.5),BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency
=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://114376524082699',Position=UDim2.fromScale(0.5,0.5),Size=UDim2.
fromScale(1,1),Visible=false,__dynamicKeys={ImageColor3=b.Theme.Controls.
WindowControlIcon[1],ImageTransparency=b.Theme.Controls.WindowControlIcon[2]}}}
return f,g end end function a.m()local c=a.cache.m if not c then c={c=b()}a.
cache.m=c end return c.c end end do local b=function()a.b()return function(b,c)
local d,e,f,g,h,i=a.i(),a.e(),a.d(),a.c(),a.k()(b)local j,k,l,m,n=a.l(),f.Create
,e.UserInputService,e.TweenService,b.__container or b.__instance or b local o,p,
q,r,s,t,u,v=k'Frame'{Name='Window',BackgroundTransparency=1,BorderSizePixel=0,
Size=UDim2.fromScale(1,1),AnchorPoint=Vector2.new(0.5,0.5),Position=UDim2.
fromScale(0.5,0.5),Parent=n}.__instance,{}c=c or{}c.Size=c.Size or UDim2.
fromOffset(850,530)c.Maximized=c.Maximized==true c.Minimized=c.Minimized==true c
.Searching=c.Searching~=false c.Resizable=c.Resizable~=false c.Draggable=c.
Draggable~=false c.CanExit=c.CanExit~=false c.CanMinimize=c.CanMinimize~=false c
.CanZoom=c.CanZoom~=false c.Resizable=c.Resizable~=false c.Dropshadow=c.
Dropshadow~=false c.UIBlur=c.UIBlur~=false c.Title=c.Title or'Title'local w,x=a.
m()(b,c)p.Body=(g.Apply(c,k'Frame'{Name='Body',BorderSizePixel=0,AnchorPoint=
Vector2.new(0.5,0.5),Position=UDim2.fromScale(0.5,0.5),Parent=o,__dynamicKeys={
BackgroundColor3=b.Theme.Controls.Sidebar[1],BackgroundTransparency=b.Theme.
Controls.Sidebar[2]},__contextKeys={BackgroundTransparency=function()return c.
UIBlur and b.Theme.Controls.Sidebar[2].Value or 0 end},k'UICorner'{Name=
'UICorner',CornerRadius=UDim.new(0,10)},k'UIListLayout'{Name='UIListLayout',
FillDirection=Enum.FillDirection.Horizontal,Padding=UDim.new(0,0),SortOrder=Enum
.SortOrder.LayoutOrder},k'Frame'{Name='Sidebar',BackgroundColor3=Color3.fromRGB(
234,234,234),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Size=UDim2.new(0,200,1,0),ClipsDescendants=true,k'UIPadding'{
Name='UIPadding'},k'Frame'{Name='Margins',BackgroundColor3=Color3.fromRGB(234,
234,234),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Size=UDim2.new(0,200,1,0),k'Folder'{Name='LayoutIgnore',k
'Frame'{Name='Shadow',AnchorPoint=Vector2.new(1,0),BorderColor3=Color3.fromRGB(0
,0,0),BorderSizePixel=0,Position=UDim2.new(1,2,0,0),Size=UDim2.new(0,5,1,0),
ZIndex=0,__dynamicKeys={BackgroundColor3=b.Theme.Controls.Separator.Shadow[1],
BackgroundTransparency=b.Theme.Controls.Separator.Shadow[2]},k'UIGradient'{Name=
'UIGradient',Transparency=NumberSequence.new{NumberSequenceKeypoint.new(0,1),
NumberSequenceKeypoint.new(1,0)}}}},k'UIPadding'{Name='UIPadding',PaddingRight=
UDim.new(0,2)},k'Frame'{Name='Toolbar',BackgroundColor3=Color3.fromRGB(255,255,
255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel
=0,Size=UDim2.new(1,0,0,52)},k'ScrollingFrame'{Name='SidebarList',
AutomaticCanvasSize=Enum.AutomaticSize.Y,BackgroundColor3=Color3.fromRGB(255,255
,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,CanvasSize=UDim2.new(),Position=UDim2.fromOffset(0,52),
ScrollBarImageColor3=Color3.fromRGB(0,0,0),ScrollBarImageTransparency=0.7,
ScrollBarThickness=6,Size=UDim2.new(1,0,1,-52),__dynamicKeys={
ScrollBarImageColor3=b.Theme.Controls.ScrollBar[1],ScrollBarImageTransparency=b.
Theme.Controls.ScrollBar[2]},k'UIListLayout'{Name='UIListLayout',Padding=UDim.
new(0,9),SortOrder=Enum.SortOrder.LayoutOrder,HorizontalAlignment=Enum.
HorizontalAlignment.Right},k'UIPadding'{Name='UIPadding',PaddingLeft=UDim.new(0,
10),PaddingRight=UDim.new(0,10),PaddingBottom=UDim.new(0,10)}}}},k'Frame'{Name=
'Separator',AnchorPoint=Vector2.new(1,0),BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,LayoutOrder=1,Position=UDim2.fromScale(1,0),Size=UDim2.new(0,1
,1,0),__dynamicKeys={BackgroundColor3=b.Theme.Controls.Separator.Background[1],
BackgroundTransparency=b.Theme.Controls.Separator.Background[2]}},k'Frame'{Name=
'ContentBody',AnchorPoint=Vector2.new(1,0),BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,ClipsDescendants=true,LayoutOrder=2,Position=UDim2.fromScale(1
,0),Size=UDim2.new(1,-201,1,0),__dynamicKeys={BackgroundColor3=b.Theme.Controls.
Background[1],BackgroundTransparency=b.Theme.Controls.Background[2]},k'UICorner'
{Name='UICorner',CornerRadius=UDim.new(0,10)},k'Frame'{Name='Content',
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(255,199,199),BorderMode=Enum.BorderMode.Inset,
BorderSizePixel=0,ClipsDescendants=true,Position=UDim2.fromOffset(0,52),Size=
UDim2.new(1,0,1,-52),k'UIPadding'{Name='Margins',PaddingBottom=UDim.new(0,3),
PaddingLeft=UDim.new(0,3),PaddingRight=UDim.new(0,3),PaddingTop=UDim.new(0,3)}},
k'Frame'{Name='CornerClip',AnchorPoint=Vector2.new(0,1),BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,Position=UDim2.fromScale(0,1),Size=UDim2.
fromOffset(10,10),ZIndex=-1,__dynamicKeys={BackgroundColor3=b.Theme.Controls.
Background[1]}}},k'Folder'{Name='LayoutIgnore',k'TextButton'{Name='TopArea',
AutoButtonColor=false,Active=false,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Size=UDim2.new(1,0,0,52),ZIndex=0,Text='',Modal=false},k'ImageLabel'{Name=
'Dropshadow',AnchorPoint=Vector2.new(0.5,0.5),BackgroundColor3=Color3.fromRGB(
255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Image='rbxassetid://138260268144845',ImageTransparency=0.65,
Interactable=false,Position=UDim2.new(0.5,0,0.5,3),ScaleType=Enum.ScaleType.
Slice,Size=UDim2.new(1,24,1,24),SliceCenter=Rect.new(28,26,96,94),ZIndex=0}}}))p
.Scale=k'UIScale'{Parent=o}p.Cornering=(p.Body:FindFirstChild'UICorner')p.
LayoutIgnore=(p.Body:FindFirstChild'LayoutIgnore')p.Dropshadow=(p.LayoutIgnore:
FindFirstChild'Dropshadow')p.Separator=(p.Body:FindFirstChild'Separator')p.
SidebarMargins=p.Body:FindFirstChild'Sidebar'p.Sidebar=(p.SidebarMargins:
FindFirstChild'Margins')p.SidebarList=(p.Sidebar:FindFirstChild'SidebarList')p.
Toolbar=(p.Sidebar:FindFirstChild'Toolbar')p.ContentBody=(p.Body:FindFirstChild
'ContentBody')p.Content=(p.ContentBody:FindFirstChild'Content')p.CornerClip=(p.
ContentBody:FindFirstChild'CornerClip')p.Titlebar=h p.Titlebar.Body.Parent=p.
ContentBody p.WindowControls=w p.WindowControls.Body.Parent=p.Toolbar q=p.Body.
Size r=p.Body.Position local y={Title=function(y)p.Titlebar.Title.Visible=y and
true or false if y then p.Titlebar.Title.Text=y end end,Subtitle=function(y)p.
Titlebar.Subtitle.Visible=y and true or false if y then p.Titlebar.Subtitle.Text
=y end end,Searching=function(y)p.Titlebar.SearchField.Body.Parent=y and p.
Titlebar.Trailing or nil p.Titlebar.SearchField.Body.Visible=y end,Draggable=
function(y)u=not y and false end,Resizable=function(y)if v and not c.Maximized
then v.SetEnabled(y)end end,Minimized=function(y,z)local A=z~=false if y and not
t then s=p.Body.Position if p.BlurModel then p.BlurModel.SetVisibility(false)end
end local B,C,D=TweenInfo.new(0.25,Enum.EasingStyle.Exponential,Enum.
EasingDirection.Out),{Position=y and UDim2.new(0.5,0,0.5,p.Body.AbsoluteSize.Y*2
)or s},{Scale=y and 0 or 1}if A then t=true m:Create(p.Body.__instance,B,C):
Play()local E=m:Create(p.Scale.__instance,B,D)E:Play()E.Completed:Once(function(
)t=false if p.BlurModel and not c.Minimized then p.BlurModel.SetVisibility(true)
end end)else p.Body.Position=C.Position p.Scale.Scale=D.Scale if y and p.
BlurModel then p.BlurModel.SetVisibility(false)end end end,Maximized=function(y,
z)z=z~=false if y then q=p.Body.Size r=p.Body.Position v.SetEnabled(false)else u
=false end local A,B,C=TweenInfo.new(0.5,Enum.EasingStyle.Exponential,Enum.
EasingDirection.Out),{Size=y and UDim2.fromScale(1,1)or q,Position=y and UDim2.
fromScale(0.5,0.5)or r},{CornerRadius=y and UDim.new(0,0)or UDim.new(0,10)}if z
then m:Create(p.Body.__instance,A,B):Play()m:Create(p.Cornering,A,C):Play()else
p.Body.Size=B.Size p.Body.Position=B.Position p.Cornering.CornerRadius=C.
CornerRadius end if not y then v.SetEnabled(c.Resizable)end end,Dropshadow=
function(y)p.Dropshadow.Visible=y end,UIBlur=function(y)if p.BlurModel then p.
BlurModel.Model:Destroy()p.BlurModel=nil end p.Body.BackgroundTransparency=y and
b.Theme.Controls.Sidebar[2].Value or 0 if y and not c.Minimized then p.BlurModel
=d.UIBlur(p.Body)end end,CanExit=function(y)c.CanExit=y p.WindowControls.Exit.
ImageColor3=x.Exit.ImageColor3()p.WindowControls.Exit.ImageTransparency=x.Exit.
ImageTransparency()p.WindowControls.Exit.Active=y end,CanMinimize=function(y)c.
CanMinimize=y p.WindowControls.Minimize.ImageColor3=x.Minimize.ImageColor3()p.
WindowControls.Minimize.ImageTransparency=x.Minimize.ImageTransparency()p.
WindowControls.Minimize.Active=y end,CanZoom=function(y)c.CanZoom=y p.
WindowControls.Zoom.ImageColor3=x.Zoom.ImageColor3()p.WindowControls.Zoom.
ImageTransparency=x.Zoom.ImageTransparency()p.WindowControls.Zoom.Active=y end}
local z=g.Wrap(c,y,p.Body)z.Type='Window'z.Theme=b.Theme z.Structures=p z.Tabs={
}z.addToHistory=i(z)z.__container=p.Content do v=j(p.Body.__instance,Vector2.
new(350,250))v.CreateClient()end do local A=true local B=function()A=not A local
B,C,D,E,F=A and UDim.new(0,0)or UDim.new(0,-200),A and UDim2.new(0,200,1,0)or
UDim2.new(0,0,1,0),A and UDim2.new(1,-201,1,0)or UDim2.new(1,0,1,0),A and UDim2.
new(0,10,1,0)or UDim2.new(0,10,0.5,0),TweenInfo.new(0.6,Enum.EasingStyle.
Exponential,Enum.EasingDirection.Out)p.Separator.Visible=A and true or false p.
CornerClip.Visible=A and true m:Create(p.ContentBody,F,{Size=D}):Play()m:Create(
p.SidebarMargins:FindFirstChild'UIPadding',F,{PaddingLeft=B}):Play()local G=m:
Create(p.SidebarMargins,F,{Size=C})m:Create(p.Titlebar.CornerClipLeft,F,{Size=E}
):Play()G:Play()if not A then G.Completed:Connect(function()p.CornerClip.Visible
=false end)end end p.Titlebar.SidebarButton.MouseButton1Click:Connect(B)end do
local A,B=p.Titlebar.SearchField.Field,{}local C,D=function(C,D)local E={}for F,
G in ipairs(D)do if G:IsA'GuiObject'then E[G]=G.Visible end end B[C]=E end,
function(C,D)local E=B[C]if not E then return end for F,G in ipairs(D)do if G:
IsA'GuiObject'then G.Visible=E[G]~=false end end end A:GetPropertyChangedSignal
'Text':Connect(function()local E,F,G={},{},p.Content:FindFirstChildOfClass
'ScrollingFrame'if not G then return end local H=G:GetDescendants()if not B[G]
then C(G,H)end local I=A.Text:lower()if I==''then D(G,H)return end for J,K in
ipairs(H)do if K:IsA'GuiObject'then local L=K:GetAttribute'SearchIndex'if
typeof(L)=='string'and L:lower():find(I)then table.insert(E,K)end end end for J,
K in ipairs(E)do F[K]=true local L=K.Parent while L and L~=G do if L:IsA
'GuiObject'then F[L]=true end L=L.Parent end end for J,K in ipairs(H)do if K:IsA
'GuiObject'then K.Visible=false end end for J,K in ipairs(E)do K.Visible=true
local L=K.Parent while L and L~=G do if L:IsA'GuiObject'then L.Visible=true end
L=L.Parent end local M=B[G]for N,O in ipairs(K:GetDescendants())do if O:IsA
'GuiObject'then local P=M and M[O]if P then O.Visible=true end end end end end)
end do local A=p.WindowControls.Body local B,C,D=(A:FindFirstChild'Exit'),(A:
FindFirstChild'Minimize'),(A:FindFirstChild'Zoom')local E=function()B:
FindFirstChild'Icon'.Visible=c.CanExit and A.GuiState~=Enum.GuiState.Hover or
false C:FindFirstChild'Icon'.Visible=c.CanMinimize and A.GuiState~=Enum.GuiState
.Hover or false D:FindFirstChild'Icon'.Visible=c.CanZoom and A.GuiState~=Enum.
GuiState.Hover or false end D.MouseButton1Click:Connect(function()if not c.
CanZoom then return end c.Maximized=not c.Maximized y.Maximized(c.Maximized,true
)end)C.MouseButton1Click:Connect(function()if not c.CanMinimize then return end
c.Minimized=not c.Minimized y.Minimized(c.Minimized,true)end)if b.Structures and
b.Structures.WindowPill then local F=b.Structures.WindowPill F.MouseButton1Click
:Connect(function()c.Minimized=not c.Minimized y.Minimized(c.Minimized,true)end)
end B.MouseButton1Click:Connect(function()if not c.CanExit then return end y.
Minimized(true)task.delay(0.25,p.Body.Destroy)end)A.MouseEnter:Connect(E)A.
MouseLeave:Connect(E)end do local A,B,C,D=(p.LayoutIgnore:FindFirstChild
'TopArea')A.InputBegan:Connect(function(E)if(E.UserInputType==Enum.UserInputType
.MouseButton1 or E.UserInputType==Enum.UserInputType.Touch)and c.Draggable and
not c.Maximized then u=true B=E.Position C=p.Body.Position if D then D:
Disconnect()end D=E.Changed:Connect(function()if E.UserInputState==Enum.
UserInputState.End then u=false end end)end end)l.InputChanged:Connect(function(
E)if E.UserInputType==Enum.UserInputType.MouseMovement or E.UserInputType==Enum.
UserInputType.Touch then if u then local F=E.Position-B local G=UDim2.new(C.X.
Scale,C.X.Offset+F.X,C.Y.Scale,C.Y.Offset+F.Y)p.Body.Position=G end end end)end
g.Apply(c,z)local A=function()for A,B in pairs(p.Body:GetDescendants())do if not
pcall(function()return B.Size end)then continue end local C=B.Size B.Size=UDim2.
fromOffset(0,0)B.Size=C end end task.defer(A)task.delay(1,A)return z end end
function a.n()local c=a.cache.n if not c then c={c=b()}a.cache.n=c end return c.
c end end do local b=function()a.b()return function(b,c)local d,e,f=a.d(),a.c(),
a.e()local g,h,i,j,k=d.Create,f.TweenService,b.Type=='Window'and b.Structures.
SidebarList or b.__instance or b,{},function(g,h,i,...)local j=table.pack(...)
for k,l in ipairs(g:GetChildren())do if l:IsA'GuiObject'then table.insert(h,l:
GetPropertyChangedSignal'AbsoluteSize':Connect(function(...)i(table.unpack(j,1,j
.n))end))table.insert(h,l:GetPropertyChangedSignal'Visible':Connect(function(...
)i(table.unpack(j,1,j.n))end))end end table.insert(h,g.ChildAdded:Connect(
function(k)if k:IsA'GuiObject'then table.insert(h,k:GetPropertyChangedSignal
'AbsoluteSize':Connect(function(...)i(table.unpack(j,1,j.n))end))table.insert(h,
k:GetPropertyChangedSignal'Visible':Connect(function(...)i(table.unpack(j,1,j.n)
)end))i(table.unpack(j,1,j.n))end end))table.insert(h,g.ChildRemoved:Connect(
function(...)i(table.unpack(j,1,j.n))end))end c=c or{}c.Title=c.Title or
'Section Title'c.Disclosure=c.Disclosure~=false c.Expanded=c.Expanded~=false j.
SidebarGroup=(e.Apply(c,g'TextButton'{AutomaticSize=Enum.AutomaticSize.Y,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,ClipsDescendants=true,Size=
UDim2.new(1,0,0,23),Text='',AutoButtonColor=false,Parent=i,g'UIListLayout'{
SortOrder=Enum.SortOrder.LayoutOrder,HorizontalAlignment=Enum.
HorizontalAlignment.Right},g'Frame'{Name='SectionHeader',AutomaticSize=Enum.
AutomaticSize.Y,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Size=UDim2.fromScale(1,0),g'Frame'{Name='Disclosure',BackgroundColor3=Color3.
fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0)
,BorderSizePixel=0,FontFace=Font.new
'rbxasset://fonts/families/SourceSansPro.json',LayoutOrder=1,Size=UDim2.
fromOffset(13,14),TextColor3=Color3.fromRGB(0,0,0),TextSize=14,g'ImageLabel'{
Name='DisclosureImage',AnchorPoint=Vector2.new(0.5,0.5),BackgroundColor3=Color3.
fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0)
,BorderSizePixel=0,Image='rbxassetid://115960806571685',Position=UDim2.
fromScale(0.5,0.5),Size=UDim2.fromOffset(11,7),__dynamicKeys={ImageColor3=b.
Theme.Text.Tertiary[1],ImageTransparency=b.Theme.Text.Tertiary[2]}}},g
'UIListLayout'{FillDirection=Enum.FillDirection.Horizontal,HorizontalFlex=Enum.
UIFlexAlignment.SpaceBetween,Padding=UDim.new(0,10),SortOrder=Enum.SortOrder.
LayoutOrder},g'UIPadding'{PaddingBottom=UDim.new(0,6),PaddingLeft=UDim.new(0,8),
PaddingRight=UDim.new(0,12),PaddingTop=UDim.new(0,3)},g'TextLabel'{Name='Title',
AutomaticSize=Enum.AutomaticSize.X,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
FontFace=Font.new('rbxassetid://12187365364',Enum.FontWeight.Bold,Enum.FontStyle
.Normal),LineHeight=0,RichText=true,Size=UDim2.fromOffset(0,14),Text=
'Section Title',TextSize=12,TextTruncate=Enum.TextTruncate.AtEnd,TextXAlignment=
Enum.TextXAlignment.Left,__dynamicKeys={TextColor3=b.Theme.Text.Tertiary[1],
TextTransparency=b.Theme.Text.Tertiary[2]}}}}))j.Layout=(j.SidebarGroup:
FindFirstChild'UIListLayout')j.SectionHeader=(j.SidebarGroup:FindFirstChild
'SectionHeader')j.Title=(j.SectionHeader:FindFirstChild'Title')j.Disclosure=(j.
SectionHeader:FindFirstChild'Disclosure')j.DisclosureImage=(j.Disclosure:
FindFirstChild'DisclosureImage')local l={}local m={Title=function(m)j.Title.Text
=m end,Disclosure=function(m)j.Disclosure.Visible=m j.SidebarGroup.Active=m end,
Expanded=function(m,n)n=(n==nil and true)or false local o,p=j.SidebarGroup.
__instance,j.DisclosureImage o.AutomaticSize=Enum.AutomaticSize.None for q,r in
ipairs(l)do r:Disconnect()end table.clear(l)local q=function(q)local r=m and j.
Layout.AbsoluteContentSize.Y or 23 local s=h:Create(o,TweenInfo.new(q and 0.35
or 0,Enum.EasingStyle.Exponential,Enum.EasingDirection.Out),{Size=UDim2.new(1,0,
0,r)})s:Play()end if m then k(o,l,q,false)end q(n)h:Create(p,TweenInfo.new(n and
0.35 or 0,Enum.EasingStyle.Exponential,Enum.EasingDirection.Out),{Rotation=m and
0 or-90}):Play()end}local n=e.Wrap(c,m,j.SidebarGroup)n.Type='Section'n.Theme=b.
Theme n.Structures=j n.__window=b j.SidebarGroup.MouseButton1Click:Connect(
function()if not c.Disclosure then return end n.Expanded=not n.Expanded end)e.
Apply(c,n)return n end end function a.o()local c=a.cache.o if not c then c={c=b(
)}a.cache.o=c end return c.c end end do local b=function()a.b()return function(b
,c)local d,e=a.d(),a.c()local f=d.Create c=c or{}local g={}g.Page=(e.Apply(c,f
'ScrollingFrame'{Name='ContentList',AutomaticCanvasSize=Enum.AutomaticSize.Y,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,CanvasSize=UDim2.new(),
ClipsDescendants=false,ScrollBarImageColor3=Color3.fromRGB(0,0,0),
ScrollBarImageTransparency=0.6,ScrollBarThickness=7,Size=UDim2.fromScale(1,1),
__dynamicKeys={ScrollBarImageColor3=b.Theme.Controls.ScrollBar[1],
ScrollBarImageTransparency=b.Theme.Controls.ScrollBar[2]},f'UIListLayout'{Name=
'UIListLayout',Padding=UDim.new(0,9),SortOrder=Enum.SortOrder.LayoutOrder},f
'UIPadding'{Name='Margins',PaddingBottom=UDim.new(0,17),PaddingLeft=UDim.new(0,
17),PaddingRight=UDim.new(0,17),PaddingTop=UDim.new(0,17)}}))local h=function()
local h={}for i,j in ipairs(__SH.COMPINDEXES)do if j.Parent==g.Page.__instance
then table.insert(h,j)end end for i,j in ipairs(h)do if j.Name=='PageSection'
then local k,l=h[i-1],j:FindFirstChildOfClass'UIPadding'if l then if k and k.
Name~='PageSection'then l.PaddingTop=UDim.new(0,18)else l.PaddingTop=UDim.new(0,
0)end end end end end g.Page.ChildAdded:Connect(h)g.Page.ChildRemoved:Connect(h)
h()local i=e.Wrap(c,{Parent=function(i)g.Page.Parent=i end},g.Page)i.Type='Page'
i.Theme=b.Theme i.Structures=g if b.__window then i.__window=b.__window end i.
__container=g.Page.__instance or g.Page return i end end function a.p()local c=a
.cache.p if not c then c={c=b()}a.cache.p=c end return c.c end end do local b=
function()a.b()return function(b,c)local d,e,f=a.d(),a.c(),a.p()local g,h,i,j=d.
Create,b.Type=='Window'and b or b.__window,b.Type=='Window'and b.Structures.
SidebarList or b.Type=='Tab'and b.__instance.Parent or b.__instance or b,{}c=c
or{}c.Title=c.Title or'Label'c.Indentation=c.Indentation or 0 c.Selected=c.
Selected==true j.Body=(e.Apply(c,g'TextButton'{Name='SidebarItem',
AutoButtonColor=false,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
FontFace=Font.new'rbxasset://fonts/families/SourceSansPro.json',LayoutOrder=h.
Tabs and#h.Tabs or 1,Size=UDim2.new(1,0,0,28),Text='',TextColor3=Color3.fromRGB(
0,0,0),TextSize=14,Parent=i,__dynamicKeys={BackgroundColor3=b.Theme.Controls.
SelectionFocused[1],BackgroundTransparency=b.Theme.Controls.SelectionFocused[2]}
,__contextKeys={BackgroundTransparency=function()return c.Selected and b.Theme.
Controls.SelectionFocused[2].Value or 1 end},g'UICorner'{Name='UICorner',
CornerRadius=UDim.new(0,5)},g'UIListLayout'{Name='UIListLayout',FillDirection=
Enum.FillDirection.Horizontal,HorizontalFlex=Enum.UIFlexAlignment.SpaceBetween,
Padding=UDim.new(0,14),SortOrder=Enum.SortOrder.LayoutOrder,VerticalAlignment=
Enum.VerticalAlignment.Center},g'UIPadding'{Name='UIPadding',PaddingLeft=UDim.
new(0,10),PaddingRight=UDim.new(0,8)},g'Frame'{Name='Leading',AutomaticSize=Enum
.AutomaticSize.Y,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Size=UDim2.new(1,-34,0,0),g'TextLabel'{Name='Title',BackgroundColor3=Color3.
fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0)
,BorderSizePixel=0,FontFace=Font.new'rbxassetid://12187365364',LayoutOrder=2,
LineHeight=0,RichText=true,Size=UDim2.new(1,-22,0,20),TextSize=15,TextTruncate=
Enum.TextTruncate.AtEnd,TextXAlignment=Enum.TextXAlignment.Left,__dynamicKeys={
TextColor3=b.Theme.Text.SelectionPrimary[1],TextTransparency=b.Theme.Text.
SelectionPrimary[2]},__contextKeys={TextColor3=function()return c.Selected and b
.Theme.Text.SelectionPrimary[1].Value or b.Theme.Text.Primary[1].Value end,
TextTransparency=function()return c.Selected and b.Theme.Text.SelectionPrimary[2
].Value or b.Theme.Text.Primary[2].Value end}},g'UIListLayout'{Name=
'UIListLayout',FillDirection=Enum.FillDirection.Horizontal,Padding=UDim.new(0,2)
,SortOrder=Enum.SortOrder.LayoutOrder},g'ImageLabel'{Name='Symbol',Visible=false
,AnchorPoint=Vector2.new(0.5,0.5),BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Image='rbxassetid://127049167227557',LayoutOrder=1,Position=UDim2.fromScale(0.5,
0.5),Size=UDim2.fromOffset(20,20),__dynamicKeys={ImageColor3=b.Theme.Controls.
Selection[1],ImageTransparency=b.Theme.Controls.Selection[2]},__contextKeys={
ImageColor3=function()return c.Selected and b.Theme.Text.SelectionPrimary[1].
Value or b.Theme.Controls.Selection[1].Value end,ImageTransparency=function()
return c.Selected and b.Theme.Text.SelectionPrimary[2].Value or b.Theme.Controls
.Selection[2].Value end}}},g'Frame'{Name='Trailing',AutomaticSize=Enum.
AutomaticSize.XY,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
LayoutOrder=1,Size=UDim2.fromOffset(20,16)}}))j.Trailing=(j.Body:FindFirstChild
'Trailing')j.Leading=(j.Body:FindFirstChild'Leading')j.Title=(j.Leading:
FindFirstChild'Title')j.Symbol=(j.Leading:FindFirstChild'Symbol')j.Padding=(j.
Body:FindFirstChild'UIPadding')local k if c and c.Page then if type(c.Page)==
'table'and c.Page.Type=='Page'then k=c.Page j.Page=k.Structures.Page else j.Page
=c.Page end else k=f(b,c and c.PageProperties)j.Page=k.Structures.Page end local
l l=e.Wrap(c,{Title=function(m)j.Title.Text=m end,Icon=function(m)j.Symbol.
Visible=m and true or false if m then j.Symbol.Image=m end end,Indentation=
function(m)j.Padding.PaddingLeft+=UDim.new(0,(11*m))end,Selected=function(m)if m
then for n,o in pairs(h.Tabs)do if o.__container==l.__container then continue
end o.Selected=false end end j.Body.Transparency=m and b.Theme.Controls.
SelectionFocused[2].Value or 1 j.Title.TextColor3=m and b.Theme.Text.
SelectionPrimary[1].Value or b.Theme.Text.Primary[1].Value j.Title.
TextTransparency=m and b.Theme.Text.SelectionPrimary[2].Value or b.Theme.Text.
Primary[2].Value j.Symbol.ImageColor3=m and b.Theme.Text.SelectionPrimary[1].
Value or b.Theme.Controls.Selection[1].Value j.Symbol.ImageTransparency=m and b.
Theme.Text.SelectionPrimary[2].Value or b.Theme.Controls.Selection[2].Value j.
Page.Parent=m and h and h.__container or nil end},j.Body)l.Navigate=function(m,n
)if not n or type(n)~='table'or n.Type~='Page'then warn
'Tab:Navigate expects a Page component.'return end local o=n.Structures.Page if
l.Selected and h and h.__container then if j.Page then j.Page.Parent=nil end o.
Parent=h.__container end j.Page=o k=n end l.Type='Tab'l.Theme=b.Theme l.
Structures=j l.__container=j.Page.__instance or j.Page l.__window=b.__window e.
Apply(c,l)if h then table.insert(h.Tabs,l)if l.Selected then h.addToHistory(l.
__container)end j.Body.MouseButton1Click:Connect(function()if l.Selected then
return end l.Selected=true h.addToHistory(l.__container)end)end if b.Type=='Tab'
then l.Indentation=b.Indentation+1 end return l end end function a.q()local c=a.
cache.q if not c then c={c=b()}a.cache.q=c end return c.c end end do local b=
function()return function(b)local c=a.d()local d,e=c.Create,{}e.Body=(d'Frame'{
Name='TitleStack',AutomaticSize=Enum.AutomaticSize.XY,BackgroundColor3=Color3.
fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0)
,BorderSizePixel=0,Size=UDim2.fromOffset(0,0),d'UIListLayout'{Name=
'UIListLayout',Padding=UDim.new(0,2),SortOrder=Enum.SortOrder.LayoutOrder,
VerticalAlignment=Enum.VerticalAlignment.Center},d'UIPadding'{Name='UIPadding'},
d'TextLabel'{Name='Subtitle',AutomaticSize=Enum.AutomaticSize.XY,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,FontFace=Font.new
'rbxassetid://12187365364',LayoutOrder=1,RichText=true,Size=UDim2.new(0,0,0,14),
Text='Subtitle',TextSize=13,TextWrapped=true,TextXAlignment=Enum.TextXAlignment.
Left,Visible=false,__dynamicKeys={TextColor3=b.Theme.Text.Secondary[1],
TextTransparency=b.Theme.Text.Secondary[2]}},d'TextLabel'{Name='Title',
AutomaticSize=Enum.AutomaticSize.XY,BackgroundColor3=Color3.fromRGB(255,255,255)
,BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
FontFace=Font.new'rbxassetid://12187365364',RichText=true,Size=UDim2.new(0,0,0,
16),Text='Title',TextColor3=Color3.fromRGB(0,0,0),TextSize=15,TextTransparency=
0.15,TextTruncate=Enum.TextTruncate.AtEnd,TextWrapped=true,TextXAlignment=Enum.
TextXAlignment.Left,__dynamicKeys={TextColor3=b.Theme.Text.Primary[1],
TextTransparency=b.Theme.Text.Primary[2]}}})e.Title=(e.Body:FindFirstChild
'Title')e.Subtitle=(e.Body:FindFirstChild'Subtitle')e.Padding=(e.Body:
FindFirstChild'UIPadding')return e end end function a.r()local c=a.cache.r if
not c then c={c=b()}a.cache.r=c end return c.c end end do local b=function()a.b(
)return function(b,c)local d,e,f=a.d(),a.c(),a.r()local g,h,i=d.Create,b.
__container or b.__instance or b,f(b)c=c or{}c.Title=c.Title or'Title'local j=g
'Frame'{Name='PageSection',BackgroundTransparency=1,Size=UDim2.fromScale(1,0),
AutomaticSize=Enum.AutomaticSize.Y,Parent=h,g'UIListLayout'{Name='UIListLayout',
Padding=UDim.new(0,12),SortOrder=Enum.SortOrder.LayoutOrder},g'UIPadding'{
PaddingBottom=UDim.new(0,18)}}i.Body.Parent=j.__instance i.Padding.PaddingLeft=
UDim.new(0,8)i.Padding.PaddingRight=UDim.new(0,8)i.Title.FontFace=Font.new(i.
Title.FontFace.Family,Enum.FontWeight.SemiBold)i.Body.LayoutOrder=-1 local k=e.
Wrap(c,{Title=function(k)i.Title.Text=k end,Subtitle=function(k)i.Subtitle.
Visible=true i.Subtitle.Text=k end},j)k.Type='PageSection'k.Theme=b.Theme k.
Structures=i k.__container=j.__instance e.Apply(c,k)return k end end function a.
s()local c=a.cache.s if not c then c={c=b()}a.cache.s=c end return c.c end end
do local b=function()a.b()return function(b,c)local d,e=a.d(),a.c()local f,g,h,i
,j=d.Create,b.__container or b.__instance or b,{},{},{}c=c or{}h.Body=(e.Apply(c
,f'Frame'{Name='Form',AutomaticSize=Enum.AutomaticSize.Y,BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,Size=UDim2.fromScale(1,0),Parent=g,
__dynamicKeys={BackgroundColor3=b.Theme.Controls.View[1],BackgroundTransparency=
b.Theme.Controls.View[2]},f'UIListLayout'{Name='UIListLayout',SortOrder=Enum.
SortOrder.LayoutOrder},f'UIPadding'{Name='Margins',PaddingLeft=UDim.new(0,10),
PaddingRight=UDim.new(0,10)},f'UICorner'{Name='UICorner',CornerRadius=UDim.new(0
,6)},f'UIStroke'{Name='UIStroke',ApplyStrokeMode=Enum.ApplyStrokeMode.Border,
__dynamicKeys={Color=b.Theme.Controls.ViewBorder[1],Transparency=b.Theme.
Controls.ViewBorder[2]}}}))local k=e.Wrap(c,{},h.Body)k.Type='Form'k.Theme=b.
Theme k.Structures=h local l=function()for l,m in ipairs(j)do local n,o=m.
section.Visible,j[l+1]local p=o and o.section.Visible m.divider.Visible=n and p
==true end end h.Body.ChildAdded:Connect(function(m)if m:IsA'Frame'and m:
FindFirstChild'LayoutIgnore'then local n=m.LayoutIgnore:FindFirstChild'Divider'
if n then table.insert(i,n)table.insert(j,{section=m,divider=n})m:
GetPropertyChangedSignal'Visible':Connect(function()l()end)l()end end end)e.
Apply(c,k)return k end end function a.t()local c=a.cache.t if not c then c={c=b(
)}a.cache.t=c end return c.c end end do local b=function()a.b()return function(b
,c)local d,e=a.d(),a.c()local f,g,h=d.Create,b.__container or b.__instance or b,
{}c=c or{}h.Body=(e.Apply(c,f'Frame'{Name='Row',AutomaticSize=Enum.AutomaticSize
.Y,BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Size=UDim2.fromScale(1,0),
Parent=g,f'Frame'{Name='LeftAccessory',AutomaticSize=Enum.AutomaticSize.Y,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Size=UDim2.fromScale(0,1),f
'UIListLayout'{Name='UIListLayout',FillDirection=Enum.FillDirection.Horizontal,
HorizontalAlignment=Enum.HorizontalAlignment.Left,Padding=UDim.new(0,10),
SortOrder=Enum.SortOrder.LayoutOrder,VerticalAlignment=Enum.VerticalAlignment.
Center,Wraps=true},f'UIPadding'{Name='UIPadding',PaddingBottom=UDim.new(0,10),
PaddingRight=UDim.new(0,10),PaddingTop=UDim.new(0,10)}},f'Frame'{Name=
'RightAccessory',AnchorPoint=Vector2.new(1,0),AutomaticSize=Enum.AutomaticSize.
XY,BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,LayoutOrder=1,Position=
UDim2.fromScale(1,0),Size=UDim2.fromScale(0,1),f'UIListLayout'{Name=
'UIListLayout',FillDirection=Enum.FillDirection.Horizontal,HorizontalAlignment=
Enum.HorizontalAlignment.Right,Padding=UDim.new(0,10),SortOrder=Enum.SortOrder.
LayoutOrder,VerticalAlignment=Enum.VerticalAlignment.Center,Wraps=true},f
'UIPadding'{Name='UIPadding',PaddingBottom=UDim.new(0,10),PaddingLeft=UDim.new(0
,11),PaddingTop=UDim.new(0,10)}},f'Folder'{Name='LayoutIgnore',f'Frame'{Name=
'Divider',AnchorPoint=Vector2.new(0,1),BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Position=UDim2.fromScale(0,1),Size=UDim2.new(1,0,0,1),Visible=
false,__dynamicKeys={BackgroundColor3=b.Theme.Controls.ViewBorder[1],
BackgroundTransparency=b.Theme.Controls.ViewBorder[2]}}}}))h.LeftAccessories=(h.
Body:FindFirstChild'LeftAccessory')h.RightAccessories=(h.Body:FindFirstChild
'RightAccessory')local i=function()local i,j,k=h.LeftAccessories.Parent.
AbsoluteSize.X,h.RightAccessories.AbsoluteSize.X,75 local l=math.max(i-j,k)if i-
j<k then h.LeftAccessories.Size=UDim2.new(0,k,1,0)else h.LeftAccessories.Size=
UDim2.new(0,l,1,0)end end i()h.RightAccessories:GetPropertyChangedSignal
'AbsoluteSize':Connect(i)h.LeftAccessories.Parent:GetPropertyChangedSignal
'AbsoluteSize':Connect(i)local j=e.Wrap(c,{SearchIndex=function(j)h.Body:
SetAttribute('SearchIndex',j)end},h.Body)j.Type='Row'j.Theme=b.Theme j.
Structures=h j.Left=function()local k=table.clone(j)k.__container=h.
LeftAccessories return k end j.Right=function()local k=table.clone(j)k.
__container=h.RightAccessories return k end e.Apply(c,j)return j end end
function a.u()local c=a.cache.u if not c then c={c=b()}a.cache.u=c end return c.
c end end do local b=function()a.b()return function(b,c)local d,e=a.d(),a.c()
local f,g,h=d.Create,b.__container or b.__instance or b,{}c=c or{}c.Padding=c.
Padding or UDim.new(0,10)c.VerticalAlignment=c.VerticalAlignment or Enum.
VerticalAlignment.Center c.HorizontalAlignment=c.HorizontalAlignment or Enum.
HorizontalAlignment.Right h.Body=(e.Apply(c,f'Frame'{Name='HStack',
BackgroundTransparency=1,AutomaticSize=Enum.AutomaticSize.XY,Parent=g,f
'UIListLayout'{Name='UIListLayout',FillDirection=Enum.FillDirection.Horizontal,
SortOrder=Enum.SortOrder.LayoutOrder,Wraps=true}}))h.Layout=(h.Body:
FindFirstChild'UIListLayout')local i=e.Wrap(c,{Padding=function(i)h.Layout.
Padding=i end,HorizontalAlignment=function(i)h.Layout.HorizontalAlignment=i end,
VerticalAlignment=function(i)h.Layout.VerticalAlignment=i end},h.Body)i.Type=
'HStack'i.Theme=b.Theme i.Structures=h e.Apply(c,i)return i end end function a.v
()local c=a.cache.v if not c then c={c=b()}a.cache.v=c end return c.c end end do
local b=function()a.b()return function(b,c)local d,e=a.d(),a.c()local f,g,h=d.
Create,b.__container or b.__instance or b,{}c=c or{}c.Padding=c.Padding or UDim.
new(0,10)c.VerticalAlignment=c.VerticalAlignment or Enum.VerticalAlignment.
Center c.HorizontalAlignment=c.HorizontalAlignment or Enum.HorizontalAlignment.
Right h.Body=(e.Apply(c,f'Frame'{Name='VStack',BackgroundTransparency=1,
AutomaticSize=Enum.AutomaticSize.XY,Parent=g,f'UIListLayout'{Name='UIListLayout'
,FillDirection=Enum.FillDirection.Vertical,SortOrder=Enum.SortOrder.LayoutOrder,
Wraps=true}}))h.Layout=(h.Body:FindFirstChild'UIListLayout')local i=e.Wrap(c,{
Padding=function(i)h.Layout.Padding=i end,HorizontalAlignment=function(i)h.
Layout.HorizontalAlignment=i end,VerticalAlignment=function(i)h.Layout.
VerticalAlignment=i end},h.Body)i.Type='VStack'i.Theme=b.Theme i.Structures=h e.
Apply(c,i)return i end end function a.w()local c=a.cache.w if not c then c={c=b(
)}a.cache.w=c end return c.c end end do local b=function()a.b()return function(b
,c)local d,e=a.c(),a.r()local f=e(b)c=c or{}c.Title=c.Title or'Title'f.Body.
Parent=b.__container or b.__instance or b local g=d.Wrap(c,{Title=function(g)f.
Title.Text=g end,Subtitle=function(g)f.Subtitle.Visible=true f.Subtitle.Text=g
end},f.Body)g.Type='TitleStack'g.Theme=b.Theme g.Structures=f d.Apply(c,g)return
g end end function a.x()local c=a.cache.x if not c then c={c=b()}a.cache.x=c end
return c.c end end do local b=function()a.b()return function(b,c)local d,e=a.d()
,a.c()local f,g,h=d.Create,b.__container or b.__instance or b,{}c=c or{}c.Text=c
.Text or'Label'h.Body=(f'TextLabel'{Name='Label',AutomaticSize=Enum.
AutomaticSize.XY,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
FontFace=Font.new'rbxassetid://12187365364',RichText=true,TextSize=15,
TextTruncate=Enum.TextTruncate.AtEnd,TextWrapped=true,Parent=g,__dynamicKeys={
TextColor3=b.Theme.Text.Secondary[1],TextTransparency=b.Theme.Text.Secondary[2]}
})local i=e.Wrap(c,{},h.Body)i.Type='Label'i.Theme=b.Theme i.Structures=h e.
Apply(c,i)return i end end function a.y()local c=a.cache.y if not c then c={c=b(
)}a.cache.y=c end return c.c end end do local b=function()a.b()return function(b
,c)local d,e=a.d(),a.c()local f,g,h=d.Create,b.__container or b.__instance or b,
{}c=c or{}c.Style=c.Style or'Primary'c.Size=c.Size or UDim2.fromOffset(20,20)h.
Body=(f'ImageLabel'{Name='Image',AutomaticSize=Enum.AutomaticSize.XY,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Parent=g,__dynamicKeys={
ImageColor3=b.Theme.Text.Primary[1],ImageTransparency=b.Theme.Text.Primary[2]},
__contextKeys={ImageColor3=function()return c.Style=='Primary'and b.Theme.Text.
Primary[1].Value or b.Theme.Text.Secondary[1].Value end,ImageTransparency=
function()return c.Style=='Primary'and b.Theme.Text.Primary[2].Value or b.Theme.
Text.Secondary[2].Value end}})local i=e.Wrap(c,{Style=function(i)if i=='Primary'
then h.Body.ImageColor3=b.Theme.Text.Primary[1].Value h.Body.ImageTransparency=b
.Theme.Text.Primary[2].Value else h.Body.ImageColor3=b.Theme.Text.Secondary[1].
Value h.Body.ImageTransparency=b.Theme.Text.Secondary[2].Value end end},h.Body)i
.Type='Symbol'i.Theme=b.Theme i.Structures=h e.Apply(c,i)return i end end
function a.z()local c=a.cache.z if not c then c={c=b()}a.cache.z=c end return c.
c end end do local b=function()a.b()return function(b,c)local d,e,f=a.d(),a.c(),
a.e()local g,h,i,j,k=d.Create,f.TweenService,b.__container or b.__instance or b,
b.Theme.Controls.Toggle,{}c=c or{}c.Value=c.Value==true k.Body=(e.Apply(c,g
'Frame'{Name='Toggle',BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Size=UDim2.fromOffset(28,16),Parent=i,g'ImageButton'{Name='Switch',
AutoButtonColor=false,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Image='rbxassetid://104426531889908',ImageColor3=Color3.fromRGB(0,0,0),
ImageTransparency=0.91,Size=UDim2.fromOffset(28,16),__dynamicKeys={ImageColor3=j
.SwitchOff[1],ImageTransparency=j.SwitchOff[2]},__contextKeys={ImageColor3=
function()return c.Value and j.SwitchOn[1].Value or j.SwitchOff[1].Value end,
ImageTransparency=function()return c.Value and j.SwitchOn[2].Value or j.
SwitchOff[2].Value end},g'ImageLabel'{Name='Knob',AnchorPoint=Vector2.new(0,0.5)
,BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://99107881432922',Position=UDim2.new(0,1,0.5,0),Size=UDim2.
fromOffset(14,14),__dynamicKeys={ImageColor3=j.Knob[1],ImageTransparency=j.Knob[
2]},g'ImageLabel'{Name='Effects',AnchorPoint=Vector2.new(0.5,0.5),
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://138042641797315',Position=UDim2.fromScale(0.5,0.5),Size=UDim2.
fromOffset(16,16),__dynamicKeys={ImageColor3=j.KnobEffects[1],ImageTransparency=
j.KnobEffects[2]}}},g'UIGradient'{Name='UIGradient',Rotation=90,__dynamicKeys={
Color=j.DepthEffect[1]}}}}))k.Switch=(k.Body:FindFirstChild'Switch')k.Knob=(k.
Switch:FindFirstChild'Knob')local l local m={Value=function(m,n)n=(n==nil and
true)or false local o,p,q=TweenInfo.new(0.4,Enum.EasingStyle.Exponential,Enum.
EasingDirection.Out),{Position=m and UDim2.new(0,13,0.5,0)or UDim2.new(0,1,0.5,0
)},{ImageColor3=m and j.SwitchOn[1].Value or j.SwitchOff[1].Value,
ImageTransparency=m and j.SwitchOn[2].Value or j.SwitchOff[2].Value}if n then h:
Create(k.Knob,o,p):Play()h:Create(k.Switch,o,q):Play()else k.Knob.Position=p.
Position k.Switch.ImageColor3=q.ImageColor3 k.Switch.ImageTransparency=q.
ImageTransparency end if c.ValueChanged and n then c.Value=m task.spawn(c.
ValueChanged,l,m)end end}l=e.Wrap(c,m,k.Body)l.Type='Toggle'l.Theme=b.Theme l.
Structures=k do k.Switch.MouseButton1Click:Connect(function()l.Value=not l.Value
end)end m.Value(not not c.Value,false)e.Apply(c,l)return l end end function a.A(
)local c=a.cache.A if not c then c={c=b()}a.cache.A=c end return c.c end end do
local b=function()a.b()return function(b,c)local d,e=a.c(),a.j()(b)c=c or{}c.
Value=c.Value~=nil and c.Value or''c.Placeholder=c.Placeholder or'Placeholder'e.
Field.TextTruncate=Enum.TextTruncate.AtEnd e.Field.TextXAlignment=Enum.
TextXAlignment.Right e.Body.Parent=b.__container or b.__instance or b local f f=
d.Wrap(c,{Placeholder=function(g)e.Field.PlaceholderText=g end,Value=function(g)
e.Field.Text=g if c.ValueChanged then c.Value=g task.spawn(c.ValueChanged,f,g)
end end},e.Body)e.Field.Focused:Connect(function()e.Body.AutomaticSize=Enum.
AutomaticSize.XY end)e.Field.FocusLost:Connect(function()e.Body.AutomaticSize=
Enum.AutomaticSize.None if f.Value~=e.Field.Text then f.Value=e.Field.Text end
end)f.Type='TextField'f.Theme=b.Theme f.Structures=e d.Apply(c,f)e.Field:
GetPropertyChangedSignal'Text':Connect(function()if c.TextChanged then task.
spawn(c.TextChanged,f,e.Field.Text or'')end end)return f end end function a.B()
local c=a.cache.B if not c then c={c=b()}a.cache.B=c end return c.c end end do
local b=function()a.b()return function(b,c)local d,e=a.c(),a.e()local f,g=e.
UserInputService,a.j()(b)c=c or{}c.Placeholder=c.Placeholder or'Press a key'g.
Field.TextXAlignment=Enum.TextXAlignment.Right g.Field.TextEditable=false g.Body
.Parent=b.__container or b.__instance or b g.Body.AutomaticSize=Enum.
AutomaticSize.XY g.Body.Size=UDim2.fromOffset(0,23)g.Field.Size=UDim2.
fromOffset(0,23)g.Corner.CornerRadius=UDim.new(0,6)local h h=d.Wrap(c,{
Placeholder=function(i)g.Field.PlaceholderText=i end,Value=function(i)g.Field.
Text=i.Name if c.ValueChanged then c.Value=i task.spawn(c.ValueChanged,h,i)end
end},g.Body)f.InputBegan:Connect(function(i,j)if i.UserInputType~=Enum.
UserInputType.Keyboard then return end if i.KeyCode==c.Value and c.BindPressed
then c.BindPressed(h,i.KeyCode,false,j)end end)f.InputEnded:Connect(function(i,j
)if i.UserInputType~=Enum.UserInputType.Keyboard then return end if g.Field:
IsFocused()then h.Value=i.KeyCode g.Field:ReleaseFocus()elseif i.KeyCode==c.
Value and c.BindPressed then c.BindPressed(h,i.KeyCode,true,j)end end)h.Type=
'KeybindField'h.Theme=b.Theme h.Structures=g d.Apply(c,h)return h end end
function a.C()local c=a.cache.C if not c then c={c=b()}a.cache.C=c end return c.
c end end do local b=function()a.b()return function(b,c)local d,e=a.d(),a.c()
local f,g,h,i,j=d.Create,b.__container or b.__instance or b,b.Theme.Controls.
Slider,0,{}c=c or{}c.Value=c.Value or 0 c.Maximum=c.Maximum or 1 c.Minimum=c.
Minimum or 0 j.Body=(e.Apply(c,f'ImageLabel'{Name='Slider',Active=true,
AnchorPoint=Vector2.new(0,0.5),BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Image='rbxassetid://92966982499851',Position=UDim2.fromScale(0,0.5),Selectable=
true,Size=UDim2.fromOffset(150,4),Parent=g,__dynamicKeys={ImageColor3=h.Track[1]
,ImageTransparency=h.Track[2]},f'ImageLabel'{Name='TrackClip',BackgroundColor3=
Color3.fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,Image='rbxassetid://113661976068590',
ImageColor3=Color3.fromRGB(252,252,252),ResampleMode=Enum.ResamplerMode.
Pixelated,Size=UDim2.new(0,2,1,0),ZIndex=3,__dynamicKeys={ImageColor3=b.Theme.
Controls.View[1],ImageTransparency=b.Theme.Controls.View[2]}},f'Frame'{Name=
'Fill',BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Size=UDim2.
fromScale(0,1),__dynamicKeys={BackgroundColor3=b.Theme.Controls.Selection[1],
BackgroundTransparency=b.Theme.Controls.Selection[2]},f'ImageLabel'{Name=
'Effects',BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://82410105327406',ImageColor3=Color3.fromRGB(0,0,0),Size=UDim2.new(0
,150,1,0),ZIndex=0,__dynamicKeys={ImageColor3=h.TrackEffects[1],
ImageTransparency=h.TrackEffects[2]}},f'Frame'{Name='Thumb',Active=true,
AnchorPoint=Vector2.new(0.5,0.5),BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Position=UDim2.fromScale(1,0),Selectable=true,Size=UDim2.fromOffset(20,20),
ZIndex=2,f'ImageLabel'{Name='Background',BackgroundColor3=Color3.fromRGB(255,255
,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Image='rbxassetid://125496304036680',Selectable=true,Size=
UDim2.fromOffset(20,20),__dynamicKeys={ImageColor3=h.Thumb[1],ImageTransparency=
h.Thumb[2]},f'UIStroke'{Name='UIStroke',ApplyStrokeMode=Enum.ApplyStrokeMode.
Border,__dynamicKeys={Color=h.ThumbStroke[1],Transparency=h.ThumbStroke[2]}},f
'UICorner'{Name='UICorner',CornerRadius=UDim.new(1,0)}},f'ImageLabel'{Name=
'ThumbEffects',AnchorPoint=Vector2.new(0.5,0),BackgroundColor3=Color3.fromRGB(
255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Image='rbxassetid://85926626300527',Position=UDim2.fromScale(
0.5,0),Size=UDim2.fromOffset(22,22),ZIndex=0,__dynamicKeys={ImageColor3=h.
ThumbEffects[1],ImageTransparency=h.ThumbEffects[2]},f'UICorner'{Name='UICorner'
,CornerRadius=UDim.new(1,0)}}}}}))j.Fill=(j.Body:FindFirstChild'Fill')j.Thumb=(j
.Fill:FindFirstChild'Thumb')j.Dragger=(f'UIDragDetector'{Name='UIDragDetector',
ResponseStyle=Enum.UIDragDetectorResponseStyle.CustomOffset,
SelectionModeDragSpeed=UDim2.new(),SelectionModeRotateSpeed=0,
ActivatedCursorIcon='rbxassetid://0',CursorIcon='rbxassetid://0',Parent=j.Thumb}
)local k local l={Value=function(l)local m,n=j.Body.AbsoluteSize.X,j.Thumb.
AbsoluteSize.X local o,p,q=n/2,c.Minimum,c.Maximum local r,s=q~=p and(l-p)/(q-p)
or 0,m-n local t=o+(s*r)j.Fill.Size=UDim2.new(0,t,1,0)if c.ValueChanged then c.
Value=l task.spawn(c.ValueChanged,k,l)end end}k=e.Wrap(c,l,j.Body)k.Type=
'Slider'k.Theme=b.Theme k.Structures=j j.Dragger.DragStart:Connect(function()i=j
.Fill.AbsoluteSize.X end)j.Dragger.DragContinue:Connect(function()local m,n=j.
Body.AbsoluteSize.X,j.Thumb.AbsoluteSize.X local o=n/2 local p,q,r=o,m-o,j.
Dragger.DragUDim2.X.Offset local s=i+r local t=math.clamp(s,p,q)local u=(t-p)/(q
-p)k.Value=k.Minimum+(k.Maximum-k.Minimum)*u end)e.Apply(c,k)return k end end
function a.D()local c=a.cache.D if not c then c={c=b()}a.cache.D=c end return c.
c end end do local b=function()a.b()return function(b,c)local d,e,f=a.d(),a.c(),
a.e()local g,h,i,j,k=d.Create,f.TweenService,b.__container or b.__instance or b,
b.Theme.Controls.Button,{}c=c or{}c.State=c.State or'Primary'c.Label=c.Label or
'Label'k.Body=(e.Apply(c,g'TextButton'{Name='Button',AutoButtonColor=false,
AutomaticSize=Enum.AutomaticSize.XY,BackgroundColor3=Color3.fromRGB(255,255,255)
,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Text='',Parent=i,g
'UICorner'{Name='UICorner',CornerRadius=UDim.new(0,5)},g'Folder'{Name=
'ShadowLayers',g'Frame'{Name='Layer1',BackgroundColor3=Color3.fromRGB(255,255,
255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel
=0,Size=UDim2.fromScale(1,1),ZIndex=0,g'UICorner'{Name='UICorner',CornerRadius=
UDim.new(0,5)},g'UIStroke'{Name='UIStroke',ApplyStrokeMode=Enum.ApplyStrokeMode.
Border,Transparency=0.95,__dynamicKeys={Color=j.Shadow}}},g'Frame'{Name='Layer2'
,BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Size=UDim2.fromScale(1,1),
ZIndex=-1,g'UICorner'{Name='UICorner',CornerRadius=UDim.new(0,5)},g'UIStroke'{
Name='UIStroke',ApplyStrokeMode=Enum.ApplyStrokeMode.Border,Thickness=2,
Transparency=0.98,__dynamicKeys={Color=j.Shadow}}}},g'Folder'{Name=
'PressOverlay',g'Frame'{Name='Overlay',BackgroundColor3=Color3.fromRGB(0,0,0),
BackgroundTransparency=1,BorderSizePixel=0,Size=UDim2.fromScale(1,1),ZIndex=0,g
'UICorner'{Name='UICorner',CornerRadius=UDim.new(0,5)}}}}))k.PressOverlay=k.Body
:FindFirstChild'PressOverlay':FindFirstChild'Overlay'k.Shadows=k.Body:
FindFirstChild'ShadowLayers'k.Shadow1=k.Shadows:FindFirstChild'Layer1':
FindFirstChild'UIStroke'k.Shadow2=k.Shadows:FindFirstChild'Layer2':
FindFirstChild'UIStroke'local l,m={TextColor3=function()local l=c.State return l
=='Primary'and b.Theme.Text.SelectionPrimary[1].Value or l=='Secondary'and b.
Theme.Text.Primary[1].Value or l=='Destructive'and b.Theme.Accents.Red[1].Value
end,TextTransparency=function()local l=c.State return l=='Primary'and b.Theme.
Text.SelectionPrimary[2].Value or l=='Secondary'and b.Theme.Text.Primary[2].
Value or l=='Destructive'and b.Theme.Accents.Red[2].Value end},{Color=function()
local l=c.State return l=='Primary'and j.FillPrimary.Value or(l=='Secondary'or l
=='Destructive')and j.FillSecondary.Value end}k.Label=(g'TextLabel'{Size=UDim2.
fromScale(1,1),Name='Label',AutomaticSize=Enum.AutomaticSize.XY,BackgroundColor3
=Color3.fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,FontFace=Font.new'rbxassetid://12187365364',
RichText=true,TextSize=14,TextWrapped=true,TextXAlignment=Enum.TextXAlignment.
Center,TextYAlignment=Enum.TextYAlignment.Center,Parent=k.Body.__instance,
__dynamicKeys={TextColor3=b.Theme.Text.SelectionPrimary[1],TextTransparency=b.
Theme.Text.SelectionPrimary[2]},__contextKeys=l,g'UIPadding'{Name='UIPadding',
PaddingBottom=UDim.new(0,3),PaddingLeft=UDim.new(0,7),PaddingRight=UDim.new(0,7)
,PaddingTop=UDim.new(0,3)}})k.Fill=(g'UIGradient'{Name='UIGradient',Rotation=90,
Parent=k.Body.__instance,__dynamicKeys={Color=j.FillPrimary},__contextKeys=m})
local n,o={State=function()task.defer(function()k.Label.TextColor3=l.TextColor3(
)k.Label.TextTransparency=l.TextTransparency()k.Fill.Color=m.Color()end)end,
Label=function(n)k.Label.Text=n end}o=e.Wrap(c,n,k.Body)o.Type='Button'o.Theme=b
.Theme o.Structures=k k.Body.MouseButton1Click:Connect(function()if c.Pushed
then task.spawn(c.Pushed,o)end end)local p=function()local p=TweenInfo.new(0.12,
Enum.EasingStyle.Sine,Enum.EasingDirection.Out)h:Create(k.PressOverlay,p,{
BackgroundTransparency=1}):Play()end k.Body.MouseButton1Down:Connect(function()
local q=TweenInfo.new(0.08,Enum.EasingStyle.Cubic,Enum.EasingDirection.Out)h:
Create(k.PressOverlay,q,{BackgroundTransparency=0.8}):Play()end)k.Body.
MouseButton1Up:Connect(p)k.Body.MouseLeave:Connect(p)e.Apply(c,o)return o end
end function a.E()local c=a.cache.E if not c then c={c=b()}a.cache.E=c end
return c.c end end do local b=function()a.b()return function(b,c)local d,e,f=a.
d(),a.c(),a.j()local g,h,i,j,k,l,m,n,o=d.Create,b.__container or b.__instance or
b,b.Theme.Controls.Stepper,{},false,{},0.35,0.083,function(g)local h=tostring(g)
local i,j=string.find(h,'%.'),0 if i then j=#h-i end return j end local p,q=
function(p,q)local r,s=o(q),o(p)local t=math.max(r,s)local u='%.'..t..'f'return
string.format(u,p)end,function(p,q)local r=10^(q or 0)return math.floor(p*r+0.5)
/r end c=c or{}c.Value=c.Value or 0 c.Step=c.Step or 0.1 c.Maximum=c.Maximum or
1 c.Minimum=c.Minimum or 0 c.Fielded=c.Fielded or false j.Body=(e.Apply(c,g
'ImageLabel'{Name='Stepper',BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Image='rbxassetid://85888499115674',Size=UDim2.fromOffset(13,20),Parent=h,
__dynamicKeys={ImageColor3=i.Background[1],ImageTransparency=i.Background[2]},g
'ImageLabel'{Name='Shadow',BackgroundTransparency=1,AnchorPoint=Vector2.new(0.5,
0.5),BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://105571270097608',LayoutOrder=1,Position=UDim2.fromScale(0.5,0.5),
Size=UDim2.fromOffset(19,27),__dynamicKeys={ImageColor3=i.Dropshadow[1],
ImageTransparency=i.Dropshadow[2]}},g'ImageButton'{Name='Up',AutoButtonColor=
false,BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://133515997946294',Size=UDim2.new(1,0,0,9),Position=UDim2.
fromOffset(0,1),__dynamicKeys={ImageColor3=b.Theme.Text.Primary[1],
ImageTransparency=b.Theme.Text.Primary[2]},__contextKeys={ImageColor3=function()
if c.Value>=c.Maximum then return b.Theme.Text.Tertiary[1].Value else return b.
Theme.Text.Primary[1].Value end end,ImageTransparency=function()if c.Value>=c.
Maximum then return b.Theme.Text.Tertiary[2].Value else return b.Theme.Text.
Primary[2].Value end end}},g'ImageButton'{Name='Down',AnchorPoint=Vector2.new(0,
1),AutoButtonColor=false,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Image='rbxassetid://83836660987173',Position=UDim2.fromScale(0,1),Size=UDim2.
new(1,0,0,9),__dynamicKeys={ImageColor3=b.Theme.Text.Primary[1],
ImageTransparency=b.Theme.Text.Primary[2]},__contextKeys={ImageColor3=function()
if c.Value<=c.Minimum then return b.Theme.Text.Tertiary[1].Value else return b.
Theme.Text.Primary[1].Value end end,ImageTransparency=function()if c.Value<=c.
Minimum then return b.Theme.Text.Tertiary[2].Value else return b.Theme.Text.
Primary[2].Value end end}},g'Frame'{Name='Top',BackgroundColor3=Color3.fromRGB(
255,255,255),BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Position=UDim2
.fromOffset(0,6),Size=UDim2.new(1,0,0,4),ZIndex=0,g'UIGradient'{Name=
'UIGradient',Rotation=90,Transparency=NumberSequence.new{NumberSequenceKeypoint.
new(0,0.949),NumberSequenceKeypoint.new(1,0.949)},__dynamicKeys={Color=i.
SegmentShadow}}},g'Frame'{Name='Separator',AnchorPoint=Vector2.new(0,0.5),
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Position=UDim2.fromScale(0,
0.5),Size=UDim2.new(1,0,0,1),ZIndex=0,__dynamicKeys={BackgroundColor3=i.
Separator[1],BackgroundTransparency=i.Separator[2]}},g'Frame'{Name='Bottom',
BackgroundColor3=Color3.fromRGB(255,255,255),BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Position=UDim2.fromOffset(0,10),Size=UDim2.new(1,0,0,4),ZIndex
=0,g'UIGradient'{Name='UIGradient',Rotation=-90,Transparency=NumberSequence.new{
NumberSequenceKeypoint.new(0,0.949),NumberSequenceKeypoint.new(1,0.949)},
__dynamicKeys={Color=i.SegmentShadow}}},g'Frame'{Name='Filler',AnchorPoint=
Vector2.new(0,0.5),BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Position
=UDim2.fromScale(0,0.5),Size=UDim2.new(1,0,0,2),ZIndex=0,__dynamicKeys={
BackgroundColor3=i.Filler[1],BackgroundTransparency=i.Filler[2]}}}))j.IPlus=(j.
Body:FindFirstChild'Up')j.IMinus=(j.Body:FindFirstChild'Down')j.Field=f(b)j.
Field.Padding.PaddingBottom=UDim.new(0,2)j.Field.Padding.PaddingTop=UDim.new(0,2
)j.Field.Padding.PaddingRight=UDim.new(0,2)j.Field.Body.Size=UDim2.fromOffset(0,
23)j.Field.Field.Size=UDim2.fromOffset(0,0)j.Field.Corner.CornerRadius=UDim.new(
0,6)j.Field.FieldPadding:Destroy()local r local s={Fielded=function(s)if s then
j.Field.Body.Parent=h j.Body.Parent=j.Field.Body.__instance j.Body.LayoutOrder=1
else j.Field.Body.Parent=nil j.Body.Parent=h j.Body.LayoutOrder=0 end end,Value=
function(s)if s<=c.Minimum then j.IMinus.ImageColor3=b.Theme.Text.Tertiary[1].
Value j.IMinus.ImageTransparency=b.Theme.Text.Tertiary[2].Value j.IMinus.
Interactable=false else j.IMinus.ImageColor3=b.Theme.Text.Primary[1].Value j.
IMinus.ImageTransparency=b.Theme.Text.Primary[2].Value j.IMinus.Interactable=
true end if s>=c.Maximum then j.IPlus.ImageColor3=b.Theme.Text.Tertiary[1].Value
j.IPlus.ImageTransparency=b.Theme.Text.Tertiary[2].Value j.IPlus.Interactable=
false else j.IPlus.ImageColor3=b.Theme.Text.Primary[1].Value j.IPlus.
ImageTransparency=b.Theme.Text.Primary[2].Value j.IPlus.Interactable=true end if
j.Field then j.Field.Field.Text=p(s,c.Step)end if c.ValueChanged then c.Value=s
task.spawn(c.ValueChanged,r,s)end end}r=e.Wrap(c,s,j.Body)r.Type='Stepper'r.
Theme=b.Theme r.Structures=j r.Increment=function()assert(c.Minimum)assert(c.
Maximum)local t,u=math.max(o(r.Value),o(c.Step)),r.Value+c.Step local v=q(u,t)r.
Value=math.clamp(v,c.Minimum,c.Maximum)end r.Decrement=function()assert(c.
Minimum)assert(c.Maximum)local t,u=math.max(o(r.Value),o(c.Step)),r.Value-c.Step
local v=q(u,t)r.Value=math.clamp(v,c.Minimum,c.Maximum)end local t,u=function(t)
k=true local u=(t=='Up')and r.Increment or r.Decrement l[t]=task.spawn(function(
)if k then task.wait(m)end if not k then return end u()while k do task.wait(n)if
k then u()end end end)end,function(t)k=false if l[t]then task.cancel(l[t])l[t]=
nil end end j.IPlus.MouseButton1Down:Connect(function()r.Increment()t'Up'end)j.
IPlus.MouseButton1Up:Connect(function()u'Up'end)j.IPlus.MouseLeave:Connect(
function()u'Up'end)j.IMinus.MouseButton1Down:Connect(function()r.Decrement()t
'Down'end)j.IMinus.MouseButton1Up:Connect(function()u'Down'end)j.IMinus.
MouseLeave:Connect(function()u'Down'end)j.Body.AncestryChanged:Connect(function(
v,w)if not w then u'Up'u'Down'for x,y in pairs(l)do if typeof(y)==
'RBXScriptConnection'then y:Disconnect()end end l={}end end)j.Field.Field.
FocusLost:Connect(function(v)if v then assert(c.Minimum)assert(c.Maximum)local w
=j.Field.Field.Text local x=tonumber(w)if x and c.Fielded then c.Value=math.
clamp(x,c.Minimum,c.Maximum)if c.Value then s.Value(c.Value)end end else j.Field
.Field.Text=tostring(c.Value)end end)e.Apply(c,r)return r end end function a.F()
local c=a.cache.F if not c then c={c=b()}a.cache.F=c end return c.c end end do
local b=function()a.b()return function(b,c)local d,e=a.d(),a.c()local f,g,h,i=d.
Create,b.__container or b.__instance or b,b.Theme.Controls.RadioButtonGroup,{
RadioButtons={}}local j=function(j,k,l)i.RadioButtons[l]=(f'Frame'{Name=
'RadioButton',AutomaticSize=Enum.AutomaticSize.XY,BackgroundColor3=Color3.
fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0)
,BorderSizePixel=0,LayoutOrder=l,Parent=i.Body.__instance,f'UIListLayout'{Name=
'UIListLayout',FillDirection=Enum.FillDirection.Horizontal,HorizontalAlignment=
Enum.HorizontalAlignment.Right,Padding=UDim.new(0,6),SortOrder=Enum.SortOrder.
LayoutOrder,VerticalAlignment=Enum.VerticalAlignment.Center},f'ImageButton'{Name
='Button',BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://118333468121914',Size=UDim2.fromOffset(14,14),__dynamicKeys={
ImageColor3=h.Background[1],ImageTransparency=h.Background[2]},__contextKeys={
ImageColor3=function()return c.Value==l and b.Theme.Controls.Selection[1].Value
or h.Background[1].Value end,ImageTransparency=function()return c.Value==l and b
.Theme.Controls.Selection[2].Value or h.Background[2].Value end},f'ImageLabel'{
Name='Overlay',BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Image='rbxassetid://118333468121914',Size=UDim2.fromOffset(14,14),__dynamicKeys=
{ImageColor3=h.Overlay[1],ImageTransparency=h.Overlay[2]},f'UIGradient'{Name=
'UIGradient',Rotation=90,Transparency=NumberSequence.new{NumberSequenceKeypoint.
new(0,0),NumberSequenceKeypoint.new(1,1)}}},f'ImageLabel'{Name='Dot',AnchorPoint
=Vector2.new(0.5,0.5),BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Image='rbxassetid://118333468121914',Position=UDim2.fromScale(0.5,0.5),ScaleType
=Enum.ScaleType.Fit,Size=UDim2.fromOffset(6,6),SliceCenter=Rect.new(7,7,7,7),
__dynamicKeys={ImageColor3=h.Dot[1],ImageTransparency=h.Dot[2]},__contextKeys={
ImageTransparency=function()return c.Value==l and h.Dot[2].Value or 1 end}},f
'ImageLabel'{Name='Stroke',AnchorPoint=Vector2.new(0.5,0.5),BackgroundColor3=
Color3.fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,Image='rbxassetid://132968326823931',Position=
UDim2.fromScale(0.5,0.5),Size=UDim2.fromScale(1,1),__dynamicKeys={ImageColor3=h.
Stroke[1],ImageTransparency=h.Stroke[2]},__contextKeys={ImageTransparency=
function()return c.Value==l and 1 or h.Stroke[2].Value end}},f'ImageLabel'{Name=
'InnerShadow',AnchorPoint=Vector2.new(0.5,0.5),BackgroundColor3=Color3.fromRGB(
255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Image='rbxassetid://95115717327743',Position=UDim2.fromScale(
0.5,0.5),Size=UDim2.fromScale(1,1),__dynamicKeys={ImageColor3=h.InnerShadow[1],
ImageTransparency=h.InnerShadow[2]},__contextKeys={ImageTransparency=function()
return c.Value==l and 1 or h.InnerShadow[2].Value end}},f'ImageLabel'{Name=
'DropShadow',AnchorPoint=Vector2.new(0.5,0.5),BackgroundColor3=Color3.fromRGB(
255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Image='rbxassetid://110994967430153',Position=UDim2.new(0.5,0,
0.5,1),ScaleType=Enum.ScaleType.Fit,Size=UDim2.fromOffset(20,20),__dynamicKeys={
ImageColor3=b.Theme.Controls.Selection[1],ImageTransparency=b.Theme.Controls.
Selection[2]},__contextKeys={ImageTransparency=function()return c.Value==l and
0.76 or 1 end}}},f'TextLabel'{Name='Label',AutomaticSize=Enum.AutomaticSize.XY,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,FontFace=Font.new
'rbxassetid://12187365364',LayoutOrder=1,RichText=true,Text=k,TextSize=15,
TextTruncate=Enum.TextTruncate.AtEnd,Visible=not not k,__dynamicKeys={TextColor3
=b.Theme.Text.Primary[1],TextTransparency=b.Theme.Text.Primary[2]}}})local m=(i.
RadioButtons[l]:FindFirstChild'Button')if m and m:IsA'ImageButton'then m.
MouseButton1Click:Connect(function()if j then j.Value=l end end)end end c=c or{}
c.Options=c.Options or{}i.Body=(e.Apply(c,f'Frame'{Name='RadioButtons',
AutomaticSize=Enum.AutomaticSize.XY,BackgroundColor3=Color3.fromRGB(255,255,255)
,BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Parent=g,f'UIListLayout'{Name='UIListLayout',FillDirection=Enum.FillDirection.
Horizontal,HorizontalAlignment=Enum.HorizontalAlignment.Right,Padding=UDim.new(0
,16),SortOrder=Enum.SortOrder.LayoutOrder,VerticalAlignment=Enum.
VerticalAlignment.Center,Wraps=true}}))local k local l={Options=function(l)for m
,n in ipairs(l)do if i.RadioButtons[m]then i.RadioButtons[m]:Destroy()end j(k,n,
m)end end,Value=function(l)for m,n in ipairs(i.RadioButtons)do local o=m==l if n
then local p=n:FindFirstChild'Button'p.ImageColor3=o and b.Theme.Controls.
Selection[1].Value or h.Background[1].Value p.ImageTransparency=o and b.Theme.
Controls.Selection[2].Value or h.Background[2].Value p:FindFirstChild'Dot'.
ImageTransparency=o and h.Dot[2].Value or 1 p:FindFirstChild'Stroke'.
ImageTransparency=o and 1 or h.Stroke[2].Value p:FindFirstChild'InnerShadow'.
ImageTransparency=o and 1 or h.InnerShadow[2].Value p:FindFirstChild'DropShadow'
.ImageTransparency=o and 0.76 or 1 end end if c.ValueChanged then c.Value=l task
.spawn(c.ValueChanged,k,l)end end}k=e.Wrap(c,l,i.Body)k.Type='RadioButtonGroup'k
.Theme=b.Theme k.Structures=i k.Option=function(m,n)local o=#i.RadioButtons+1 j(
k,n,o)table.insert(c.Options or{},n)return i.RadioButtons[o]end e.Apply(c,k)
return k end end function a.G()local c=a.cache.G if not c then c={c=b()}a.cache.
G=c end return c.c end end do local b=function()local b=a.d()return function(c,d
)local e=b.Create d=d or{}local f,g=c.Theme.Controls.MenuButton,{}g.Container=(e
'ScreenGui'{Name='Container',IgnoreGuiInset=true,ResetOnSpawn=false,ScreenInsets
=Enum.ScreenInsets.None,ZIndexBehavior=Enum.ZIndexBehavior.Sibling,DisplayOrder=
201,OnTopOfCoreBlur=true,e'CanvasGroup'{Name='MenuBody',BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,BackgroundTransparency=1,GroupTransparency=1,
Size=UDim2.fromScale(1,1),e'ScrollingFrame'{Name=d.Name or'DropdownMenu',
AutomaticSize=Enum.AutomaticSize.XY,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,AutomaticCanvasSize=Enum.AutomaticSize.Y,CanvasSize=UDim2.new(
),ScrollBarImageColor3=Color3.fromRGB(0,0,0),ScrollBarImageTransparency=0.5,
ScrollBarThickness=3,__dynamicKeys={BackgroundColor3=f.MenuBackground[1],
BackgroundTransparency=f.MenuBackground[2],ScrollBarImageColor3=c.Theme.Controls
.ScrollBar[1],ScrollBarImageTransparency=c.Theme.Controls.ScrollBar[2]},e
'UICorner'{Name='UICorner',CornerRadius=UDim.new(0,6)},e'UIStroke'{Name=
'UIStroke',ApplyStrokeMode=Enum.ApplyStrokeMode.Border,Transparency=0.9},e
'UIPadding'{Name='UIPadding',PaddingBottom=UDim.new(0,5),PaddingLeft=UDim.new(0,
5),PaddingRight=UDim.new(0,5),PaddingTop=UDim.new(0,5)},e'UIListLayout'{Name=
'UIListLayout',SortOrder=Enum.SortOrder.LayoutOrder,Padding=UDim.new(0,1)}}}})g.
Body=(g.Container:FindFirstChild'MenuBody')g.Menu=(g.Body:FindFirstChild(d.Name
or'DropdownMenu'))function g.Item(h,i,j)local k k=(e'TextButton'{Name='Item',
Size=UDim2.fromScale(1,0),AutoButtonColor=false,AutomaticSize=Enum.AutomaticSize
.XY,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,BackgroundTransparency=
1,Selectable=false,Text='',LayoutOrder=i,Parent=g.Menu,__dynamicKeys={
BackgroundColor3=c.Theme.Controls.SelectionFocused[1]},__contextKeys={
BackgroundTransparency=function()if not k then return end return k.GuiState==
Enum.GuiState.Hover and c.Theme.Controls.SelectionFocused[2].Value or 1 end},e
'UIPadding'{Name='UIPadding',PaddingBottom=UDim.new(0,3),PaddingLeft=UDim.new(0,
7),PaddingRight=UDim.new(0,12),PaddingTop=UDim.new(0,3)},e'UIListLayout'{Name=
'UIListLayout',FillDirection=Enum.FillDirection.Horizontal,Padding=UDim.new(0,2)
,SortOrder=Enum.SortOrder.LayoutOrder,VerticalAlignment=Enum.VerticalAlignment.
Center},e'TextLabel'{Name='Label',AutomaticSize=Enum.AutomaticSize.XY,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,FontFace=Font.new
'rbxassetid://12187365364',LayoutOrder=1,RichText=true,Text=h or'',TextColor3=
Color3.fromRGB(0,0,0),TextSize=15,TextTransparency=0.15,TextTruncate=Enum.
TextTruncate.AtEnd,__dynamicKeys={TextColor3=c.Theme.Text.Primary[1],
TextTransparency=c.Theme.Text.Primary[2]}},e'UICorner'{Name='UICorner',
CornerRadius=UDim.new(0,5)}})if j then e'ImageLabel'{Name='Checkmark',
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://127464789357538',ImageColor3=Color3.fromRGB(0,0,0),
ImageTransparency=1,Size=UDim2.fromOffset(16,16),Visible=false,Parent=k.
__instance,__dynamicKeys={ImageColor3=c.Theme.Text.Primary[1],ImageTransparency=
c.Theme.Text.Primary[2]}}e'Frame'{Name='CheckmarkRepl',BackgroundColor3=Color3.
fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0)
,BorderSizePixel=0,Size=UDim2.fromOffset(16,16),Visible=true,Parent=k.__instance
}end return k.__instance end return g end end function a.H()local c=a.cache.H if
not c then c={c=b()}a.cache.H=c end return c.c end end do local b=function()
local b,c,d,e,f,g,h=a.H(),a.e(),a.a(),6,-1,6,{}h.__index=h local i,j,k,l,m,n,o,p
=c.UserInputService,c.TweenService,c.Workspace,c.GuiService,function(i)if
typeof(i)=='table'then local j,k=pcall(function()return i.__instance end)if j
and typeof(k)=='Instance'then return k end end return i end,function(i,j,k)if
not i then return false end local l,m=i.AbsolutePosition,i.AbsoluteSize return j
>=l.X and j<=l.X+m.X and k>=l.Y and k<=l.Y+m.Y end,function(i)if typeof(i)==
'table'then return i[1]end if typeof(i)=='number'then return i end return nil
end,function(i,j)if typeof(i)~='table'then return nil end return i[j]end local q
=function(q)local r=m(q)if typeof(r)~='Instance'then return false end if r.
Parent then return true end return typeof(d.ProtectUI(r))=='Instance'end
function h.new(r,s,t)t=t or{}local u=b(r,{Name=t.Name})s.MenuContainer=u.
Container.__instance s.MenuBody=u.Body s.Menu=u.Menu s.Options=s.Options or{}
local v=setmetatable({Context=r,Structures=s,MenuStructures=u,Object=nil,Anchor=
t.Anchor,AnchorMode=t.AnchorMode or'BelowButton',Gap=t.Gap or g,MaxHeight=t.
MaxHeight,ShowsCheckmarks=t.ShowsCheckmarks==true,AutoScroll=t.AutoScroll~=false
,OnSelected=t.OnSelected,SelectedValue=nil,Tween=nil,TweenToken=0,Bound=false},h
)return v end function h:SetObject(r)self.Object=r self:Bind()end function h:
SetAnchor(r)self.Anchor=r self:AnchorMenu()end function h:SetAnchorMode(r)self.
AnchorMode=r self:AnchorMenu()end function h:SetValue(r)self.SelectedValue=r
self:AnchorMenu()end function h:GetAnchorElement()local r=self.Anchor local s=m(
r)if typeof(s)=='Instance'then return s end local t=p(r,'Object')or p(r,
'Element')or p(r,'Label')or self.Structures.CurrentTab or self.Structures.Body
return m(t)end function h:GetAnchorOption()return p(self.Anchor,'Option')or o(
self.SelectedValue)or 1 end function h:GetAnchorOffsets()local r=self.Anchor
local s,t,u=p(r,'Offset'),p(r,'XOffset')or 0,p(r,'YOffset')or 0 if typeof(s)==
'Vector2'then t+=s.X u+=s.Y end return t,u end function h:GetOptionLabel(r)local
s=self.Structures.Options[r]if not s then return nil end return(s:FindFirstChild
'Label')end function h:ScrollToAnchorOption()local r=self.Structures.Menu if not
r then return end local s,t,u=self.Structures.Options[self:GetAnchorOption()],r.
AbsoluteCanvasSize.Y,r.AbsoluteSize.Y local v=math.max(0,t-u)if not s or v<=0
then r.CanvasPosition=Vector2.new(r.CanvasPosition.X,0)return end local w,x=s.
AbsolutePosition.Y-r.AbsolutePosition.Y+r.CanvasPosition.Y,s.AbsoluteSize.Y
local y=w-((u-x)/2)r.CanvasPosition=Vector2.new(r.CanvasPosition.X,math.clamp(y,
0,v))end function h:Resize()local r=self.Structures.Menu if not r then return
end r.AutomaticSize=Enum.AutomaticSize.XY if not self.MaxHeight then return end
task.defer(function()if not r then return end local s=r.AbsoluteSize.Y if s>=
self.MaxHeight then r.AutomaticSize=Enum.AutomaticSize.X r.Size=UDim2.
fromOffset(0,self.MaxHeight)else r.Size=UDim2.fromOffset(0,s)r.AutomaticSize=
Enum.AutomaticSize.X end self:AnchorMenu()end)end function h:AnchorMenu()local r
,s=m(self.Structures.Body),self.Structures.Menu local t=s and s.Parent if not s
or not r or not t then return end local u=k.CurrentCamera if not u then return
end if self.AutoScroll then self:ScrollToAnchorOption()end local v,w,x,y,z,A=r.
AbsolutePosition,r.AbsoluteSize,s.AbsolutePosition,t.AbsolutePosition,s.
AbsoluteSize,u.ViewportSize local B,C,D,E=v.X+w.X-z.X+f,v.Y+w.Y+self.Gap,self:
GetAnchorOffsets()if self.AnchorMode=='SelectedLabel'then local F,G=self:
GetAnchorElement(),self:GetOptionLabel(self:GetAnchorOption())if F and G then
local H=G.AbsolutePosition-x B=F.AbsolutePosition.X-H.X C=F.AbsolutePosition.Y-H
.Y elseif F then B=F.AbsolutePosition.X+f C=F.AbsolutePosition.Y end end B+=D C
+=E local F,G=math.max(e,A.X-e-z.X),math.max(e,A.Y-e-z.Y)B=math.clamp(B,e,F)C=
math.clamp(C,e,G)s.Position=UDim2.fromOffset(B-y.X,C-y.Y)end function h:
SetExpanded(r)local s,t,u=r==true,m(self.Structures.Body),m(self.Structures.
MenuContainer)self.TweenToken+=1 local v=self.TweenToken if self.Tween then self
.Tween:Cancel()self.Tween=nil end if s and t then self.Structures.MenuBody.
GroupTransparency=1 if not q(self.Structures.MenuContainer)then return end
elseif not s and u and not u.Parent then self.Structures.MenuBody.
GroupTransparency=1 return end self:Resize()self:AnchorMenu()task.defer(function
()self:AnchorMenu()end)local w,x={GroupTransparency=s and 0 or 1},TweenInfo.new(
s and 0 or 0.4,Enum.EasingStyle.Exponential)local y=j:Create(self.Structures.
MenuBody,x,w)self.Tween=y y:Play()y.Completed:Connect(function(z)if v~=self.
TweenToken then return end if self.Tween==y then self.Tween=nil end if z~=Enum.
PlaybackState.Completed then return end if not s and self.Object and self.Object
.Expanded==false then self.Structures.MenuContainer.Parent=nil end end)end
function h:SetSelectedMap(r)if not self.ShowsCheckmarks then return end for s,t
in ipairs(self.Structures.Options)do if not t then continue end local u,v,w=t:
FindFirstChild'Checkmark',t:FindFirstChild'CheckmarkRepl',r[s]==true if u then u
.Visible=w end if v then v.Visible=not w end end end function h:ResetItemColors(
r)local s,t=r:FindFirstChild'Label',r:FindFirstChild'Checkmark'r.
BackgroundTransparency=1 if s then s.TextColor3=self.Context.Theme.Text.Primary[
1].Value s.TextTransparency=self.Context.Theme.Text.Primary[2].Value end if t
then t.ImageColor3=self.Context.Theme.Text.Primary[1].Value t.ImageTransparency=
self.Context.Theme.Text.Primary[2].Value end end function h:HighlightItem(r)
local s,t=r:FindFirstChild'Label',r:FindFirstChild'Checkmark'r.
BackgroundTransparency=self.Context.Theme.Controls.SelectionFocused[2].Value if
s then s.TextColor3=self.Context.Theme.Controls.SelectionFocusedAccent[1].Value
s.TextTransparency=self.Context.Theme.Controls.SelectionFocusedAccent[2].Value
end if t then t.ImageColor3=self.Context.Theme.Controls.SelectionFocusedAccent[1
].Value t.ImageTransparency=self.Context.Theme.Controls.SelectionFocusedAccent[2
].Value end end function h:AddOption(r,s)local t=self.MenuStructures.Item(r,s,
self.ShowsCheckmarks)self.Structures.Options[s]=t t:GetPropertyChangedSignal
'GuiState':Connect(function()if not self.Object or not self.Object.Expanded then
return end if t.GuiState==Enum.GuiState.Hover then self:HighlightItem(t)else
self:ResetItemColors(t)end end)t.MouseButton1Click:Connect(function()if self.
OnSelected and self.Object then self.OnSelected(self.Object,s)end end)self:
Resize()task.defer(function()self:AnchorMenu()end)return t end function h:
ClearOptions()for r,s in ipairs(self.Structures.Options)do s:Destroy()end table.
clear(self.Structures.Options)self:Resize()end function h:RemoveOption(r)if r
and self.Structures.Options[r]then self.Structures.Options[r]:Destroy()self.
Structures.Options[r]=nil end self:Resize()end function h:Bind()if self.Bound
then return end self.Bound=true self.Structures.Body:GetPropertyChangedSignal
'AbsolutePosition':Connect(function()self:AnchorMenu()end)self.Structures.Body:
GetPropertyChangedSignal'AbsoluteSize':Connect(function()self:AnchorMenu()end)
self.Structures.Menu:GetPropertyChangedSignal'AbsoluteSize':Connect(function()
self:AnchorMenu()end)i.InputBegan:Connect(function(r)if not self.Object or not
self.Object.Expanded or(r.UserInputType~=Enum.UserInputType.MouseButton1 and r.
UserInputType~=Enum.UserInputType.Touch)then return end local s,t=i:
GetMouseLocation(),l:GetGuiInset()local u,v=s.Y-t.Y,m(self.Structures.Body)local
w=n(v,s.X,u)or n(v,s.X,s.Y)if not n(self.Structures.Menu,s.X,u)and not w then
self.Object.Expanded=false end end)self.Structures.Body.MouseButton1Click:
Connect(function()if self.Object then self.Object.Expanded=not self.Object.
Expanded end end)end return h end function a.I()local c=a.cache.I if not c then
c={c=b()}a.cache.I=c end return c.c end end do local b=function()a.b()return
function(b,c)local d,e,f=a.d(),a.c(),a.I()local g,h,i,j,k=d.Create,30,b.
__container or b.__instance or b,b.Theme.Controls.MenuButton,{Options={}}local l
,m=function(l)if#l>h then return string.sub(l,1,h)..'...'end return l end,
function(l)local m={}if typeof(l)=='table'then for n,o in ipairs(l)do m[o]=true
end elseif l then m[l]=true end return m end local n=function(n)local o,p={},m(n
)for q,r in ipairs(k.Options)do local s=r and r:FindFirstChild'Label'if p[q]and
s then table.insert(o,s.Text)end end return o end c=c or{}c.Maximum=c.Maximum or
1 c.Expanded=c.Expanded or false c.Options=c.Options or{}k.Body=(e.Apply(c,g
'TextButton'{Name='PopUpButton',AutomaticSize=Enum.AutomaticSize.XY,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Selectable=false,Text='',
Parent=i,g'UIPadding'{Name='UIPadding',PaddingBottom=UDim.new(0,3),PaddingLeft=
UDim.new(0,3),PaddingRight=UDim.new(0,3),PaddingTop=UDim.new(0,3)},g
'UIListLayout'{Name='UIListLayout',FillDirection=Enum.FillDirection.Horizontal,
Padding=UDim.new(0,7),SortOrder=Enum.SortOrder.LayoutOrder,VerticalAlignment=
Enum.VerticalAlignment.Center},g'Frame'{Name='PopUpIndicator',BorderColor3=
Color3.fromRGB(0,0,0),BorderSizePixel=0,LayoutOrder=1,Selectable=true,Size=UDim2
.fromOffset(16,16),__dynamicKeys={BackgroundColor3=j.IndicatorBackground[1],
BackgroundTransparency=j.IndicatorBackground[2]},g'UICorner'{Name='UICorner',
CornerRadius=UDim.new(0,4)},g'ImageLabel'{Name='Indicators',BackgroundColor3=
Color3.fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,Image='rbxassetid://89151647333378',Size=UDim2.
fromScale(1,1),__dynamicKeys={ImageColor3=b.Theme.Text.Primary[1],
ImageTransparency=b.Theme.Text.Primary[2]}}},g'TextLabel'{Name='Label',
AutomaticSize=Enum.AutomaticSize.XY,BackgroundColor3=Color3.fromRGB(255,255,255)
,BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
FontFace=Font.new'rbxassetid://12187365364',RichText=true,Text='None',TextSize=
15,TextTransparency=0.15,TextTruncate=Enum.TextTruncate.AtEnd,__dynamicKeys={
TextColor3=b.Theme.Text.Primary[1],TextTransparency=b.Theme.Text.Primary[2]}}}))
k.PopUpIndicator=(k.Body:FindFirstChild'PopUpIndicator')k.CurrentTab=(k.Body:
FindFirstChild'Label')local o,p=(f.new(b,k,{Name='PopUpMenu',Anchor=c.Anchor,
AnchorMode='SelectedLabel',ShowsCheckmarks=true,MaxHeight=239,OnSelected=
function(o,p)if o.Maximum and o.Maximum>1 then local q=o.Value or{}if typeof(q)
~='table'then q=q and{q}or{}else q=table.clone(q)end local r for s,t in ipairs(q
)do if t==p then r=s break end end if r then table.remove(q,r)else if#q<o.
Maximum then table.insert(q,p)else table.remove(q,1)table.insert(q,p)end end o.
Value=q else o.Expanded=false task.wait(0.2)o.Value=p end end}))local q=function
(q,r)local s=p and p.Maximum and p.Maximum>1 o:SetAnchorMode(s and'BelowButton'
or'SelectedLabel')o:SetValue(q)if s then local t=n(q)if#t==0 then k.CurrentTab.
Text='None'else k.CurrentTab.Text=l(table.concat(t,', '))end o:SetSelectedMap(m(
q))else local t=q and k.Options[q]and k.Options[q]:FindFirstChild'Label'if not t
then k.CurrentTab.Text='None'else k.CurrentTab.Text=l(t.Text)end o:
SetSelectedMap(m(q))end if r and c.ValueChanged then c.Value=q task.spawn(c.
ValueChanged,p,q)end end local r={Anchor=function(r)o:SetAnchor(r)end,Options=
function(r)o:ClearOptions()for s,t in ipairs(r)do o:AddOption(t,s)end if p then
q(p.Value,false)end end,Expanded=function(r)o:SetAnchorMode(p and p.Maximum and
p.Maximum>1 and'BelowButton'or'SelectedLabel')o:SetExpanded(r)end,Value=function
(r)q(r,true)end}p=e.Wrap(c,r,k.Body)o:SetObject(p)p.Type='PopUpButton'p.Theme=b.
Theme p.Structures=k p.Option=function(s,t)local u=#k.Options+1 local v=o:
AddOption(t,u)table.insert(p.Options,t)q(p.Value,false)return v end p.Remove=
function(s,t)if t then o:RemoveOption(t)p.Options[t]=nil q(p.Value,false)end end
e.Apply(c,p)q(p.Value,false)return p end end function a.J()local c=a.cache.J if
not c then c={c=b()}a.cache.J=c end return c.c end end do local b=function()a.b(
)return function(b,c)local d,e,f=a.d(),a.c(),a.I()local g,h,i,j=d.Create,b.
__container or b.__instance or b,b.Theme.Controls.MenuButton,{Options={}}c=c or{
}c.Expanded=c.Expanded or false c.Options=c.Options or{}j.Body=(e.Apply(c,g
'TextButton'{Name='PullDownButton',AutomaticSize=Enum.AutomaticSize.XY,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Selectable=false,Text='',
Parent=h,g'UIPadding'{Name='UIPadding',PaddingBottom=UDim.new(0,3),PaddingLeft=
UDim.new(0,7),PaddingRight=UDim.new(0,3),PaddingTop=UDim.new(0,3)},g
'UIListLayout'{Name='UIListLayout',FillDirection=Enum.FillDirection.Horizontal,
Padding=UDim.new(0,7),SortOrder=Enum.SortOrder.LayoutOrder,VerticalAlignment=
Enum.VerticalAlignment.Center},g'Frame'{Name='PullDownIndicator',BorderColor3=
Color3.fromRGB(0,0,0),BorderSizePixel=0,LayoutOrder=1,Selectable=true,Size=UDim2
.fromOffset(16,16),__dynamicKeys={BackgroundColor3=i.IndicatorBackground[1],
BackgroundTransparency=i.IndicatorBackground[2]},g'UICorner'{Name='UICorner',
CornerRadius=UDim.new(0,4)},g'ImageLabel'{Name='Indicators',BackgroundColor3=
Color3.fromRGB(255,255,255),BackgroundTransparency=1,BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,Image='rbxassetid://86693411280110',Size=UDim2.
fromScale(1,1),__dynamicKeys={ImageColor3=b.Theme.Text.Primary[1],
ImageTransparency=b.Theme.Text.Primary[2]}}},g'TextLabel'{Name='Label',
AutomaticSize=Enum.AutomaticSize.XY,BackgroundColor3=Color3.fromRGB(255,255,255)
,BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
FontFace=Font.new'rbxassetid://12187365364',RichText=true,TextSize=15,
TextTransparency=0.15,TextTruncate=Enum.TextTruncate.AtEnd,Visible=false,
__dynamicKeys={TextColor3=b.Theme.Text.Primary[1],TextTransparency=b.Theme.Text.
Primary[2]}}}))j.PullDownIndicator=(j.Body:FindFirstChild'PullDownIndicator')j.
CurrentTab=(j.Body:FindFirstChild'Label')local k,l=(f.new(b,j,{Name=
'PullDownMenu',Anchor=c.Anchor,AnchorMode='BelowButton',Gap=6,MaxHeight=239,
AutoScroll=false,OnSelected=function(k,l)k.Expanded=false k.Value=l end}))local
m={Anchor=function(m)k:SetAnchor(m)end,Options=function(m)k:ClearOptions()for n,
o in ipairs(m)do k:AddOption(o,n)end end,Expanded=function(m)k:SetExpanded(m)end
,Label=function(m)if type(m)=='string'then j.CurrentTab.Visible=true j.
CurrentTab.Text=m end end,Value=function(m)k:SetValue(m)if c.ValueChanged then c
.Value=m task.spawn(c.ValueChanged,l,m)end end}l=e.Wrap(c,m,j.Body)k:SetObject(l
)l.Type='PullDownButton'l.Theme=b.Theme l.Structures=j l.Option=function(n,o)
local p=#j.Options+1 local q=k:AddOption(o,p)table.insert(l.Options,o)return q
end l.Remove=function(n,o)if o then k:RemoveOption(o)l.Options[o]=nil end end e.
Apply(c,l)k:SetValue(l.Value)return l end end function a.K()local c=a.cache.K if
not c then c={c=b()}a.cache.K=c end return c.c end end do local b=function()
local b,c={},a.e()local d,e,f,g,h=c.TweenService,{},4,-12,0.05 function b.
AddNotification(i,j)if#e>=f then local k=table.remove(e,1)if k and k.Object and
k.Object.Close then k.Object:Close()end end table.insert(e,{Object=i,Frame=j})b.
UpdateStack()end function b.UpdateStack()local i=#e for j=1,i do local k=e[j]
local l=k.Frame.__instance if not l or not l.Parent then continue end local m=i-
j local n,o=1-(m*h),(m*g)l.ZIndex=100-m local p,q=-162.5,0 local r,s=q+o,l:
FindFirstChild'UIScale'if m>0 then if s then d:Create(s,TweenInfo.new(0.4,Enum.
EasingStyle.Exponential,Enum.EasingDirection.Out),{Scale=n}):Play()end d:Create(
l,TweenInfo.new(0.4,Enum.EasingStyle.Exponential,Enum.EasingDirection.Out),{
Position=UDim2.new(1,p,1,r)}):Play()if k.Object.UpdateState then k.Object:
UpdateState(m)end else if s then d:Create(s,TweenInfo.new(0.4,Enum.EasingStyle.
Exponential,Enum.EasingDirection.Out),{Scale=1}):Play()end d:Create(l,TweenInfo.
new(0.4,Enum.EasingStyle.Exponential,Enum.EasingDirection.Out),{Position=UDim2.
new(1,p,1,q)}):Play()if k.Object.UpdateState then k.Object:UpdateState(m)end end
end end function b.RemoveNotification(i)for j,k in ipairs(e)do if k.Object==i
then table.remove(e,j)break end end b.UpdateStack()end return b end function a.L
()local c=a.cache.L if not c then c={c=b()}a.cache.L=c end return c.c end end do
local b=function()a.b()return function(b,c,d,e)local f=a.d()local g,h=f.Create,b
.__instance:FindFirstChild'Notifications'if not h then h=g'Frame'{Name=
'Notifications',BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Size=UDim2.fromScale(1,1),ZIndex=200,Parent=b.__instance,g'UIPadding'{Name=
'UIPadding',PaddingBottom=UDim.new(0,15),PaddingRight=UDim.new(0,15)}}.
__instance end local i=(g'Frame'{Name='Notification',AnchorPoint=Vector2.new(0.5
,1),AutomaticSize=Enum.AutomaticSize.Y,BackgroundTransparency=1,BorderColor3=
Color3.fromRGB(0,0,0),BorderSizePixel=0,Position=UDim2.new(1,-162.5,1,0),Size=
UDim2.fromOffset(325,0),Parent=h,g'Frame'{Name='Canvas',AnchorPoint=Vector2.new(
0,0),AutomaticSize=Enum.AutomaticSize.Y,BackgroundColor3=b.Theme.Controls.
Sidebar[1].Value,BackgroundTransparency=0.1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Position=UDim2.fromScale(0,0),Size=UDim2.fromScale(1,0),
__dynamicKeys={BackgroundColor3=b.Theme.Controls.Sidebar[1]},__contextKeys={
BackgroundTransparency=function()return d.Age>=(e-1)and 1 or 0.1 end},g
'UICorner'{Name='UICorner',CornerRadius=UDim.new(0,12)},g'UIListLayout'{Name=
'UIListLayout',Padding=UDim.new(0,5),SortOrder=Enum.SortOrder.LayoutOrder,
VerticalAlignment=Enum.VerticalAlignment.Center},g'Frame'{Name='Content',
AutomaticSize=Enum.AutomaticSize.Y,BackgroundColor3=Color3.fromRGB(255,0,68),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
LayoutOrder=1,Size=UDim2.fromScale(1,0),g'Frame'{Name='TitleContainer',
AutomaticSize=Enum.AutomaticSize.XY,BackgroundColor3=Color3.fromRGB(255,255,255)
,BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
LayoutOrder=1,g'UIListLayout'{Name='UIListLayout',FillDirection=Enum.
FillDirection.Horizontal,Padding=UDim.new(0,5),SortOrder=Enum.SortOrder.
LayoutOrder,VerticalAlignment=Enum.VerticalAlignment.Center},g'ImageLabel'{Name=
'Icon',BackgroundTransparency=1,LayoutOrder=1,Size=UDim2.fromOffset(18,18),
Visible=false,__dynamicKeys={ImageColor3=b.Theme.Text.Primary[1]},__contextKeys=
{ImageTransparency=function()return d.Age>=(e-1)and 1 or b.Theme.Text.Primary[2]
.Value end}},g'TextLabel'{Name='Title',AutomaticSize=Enum.AutomaticSize.Y,
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,LayoutOrder=2,FontFace=Font
.new('rbxassetid://12187365364',Enum.FontWeight.Bold,Enum.FontStyle.Normal),
LineHeight=0,Size=UDim2.new(1,0,0,20),Text=c.Title,TextColor3=b.Theme.Text.
Primary[1].Value,TextSize=13,TextTransparency=b.Theme.Text.Primary[2].Value,
TextWrapped=true,RichText=true,TextXAlignment=Enum.TextXAlignment.Left,
__dynamicKeys={TextColor3=b.Theme.Text.Primary[1],TextTransparency=b.Theme.Text.
Primary[2]},__contextKeys={TextTransparency=function()return d.Age>=(e-1)and 1
or b.Theme.Text.Primary[2].Value end}}},g'UIListLayout'{Name='UIListLayout',
SortOrder=Enum.SortOrder.LayoutOrder},g'TextLabel'{Name='Subtitle',AutomaticSize
=Enum.AutomaticSize.Y,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
FontFace=Font.new'rbxassetid://12187365364',LayoutOrder=1,Position=UDim2.
fromScale(0,0.147),RichText=true,Size=UDim2.fromScale(1,0),Text=c.Subtitle,
TextColor3=b.Theme.Text.Secondary[1].Value,TextSize=13,TextTransparency=b.Theme.
Text.Secondary[2].Value,TextWrapped=true,TextXAlignment=Enum.TextXAlignment.Left
,Visible=c.Subtitle~='',__dynamicKeys={TextColor3=b.Theme.Text.Secondary[1],
TextTransparency=b.Theme.Text.Secondary[2]},__contextKeys={TextTransparency=
function()return d.Age>=(e-1)and 1 or b.Theme.Text.Secondary[2].Value end}},g
'UIPadding'{Name='UIPadding',PaddingBottom=UDim.new(0,12),PaddingLeft=UDim.new(0
,12),PaddingRight=UDim.new(0,12)}},g'Frame'{Name='Topbar',AutomaticSize=Enum.
AutomaticSize.Y,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
Size=UDim2.fromScale(1,0),Visible=(c.App~=nil)or(c.AppIcon~=nil),g'UIListLayout'
{Name='UIListLayout',FillDirection=Enum.FillDirection.Horizontal,Padding=UDim.
new(0,5),SortOrder=Enum.SortOrder.LayoutOrder,VerticalAlignment=Enum.
VerticalAlignment.Center},g'TextLabel'{Name='App',AutomaticSize=Enum.
AutomaticSize.X,BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
FontFace=Font.new('rbxassetid://12187365364',Enum.FontWeight.Medium,Enum.
FontStyle.Normal),LayoutOrder=1,LineHeight=0,Size=UDim2.fromOffset(0,14),Text=c.
App or'',TextColor3=b.Theme.Text.Primary[1].Value,TextSize=13,TextTransparency=
0.5,TextXAlignment=Enum.TextXAlignment.Left,RichText=true,Visible=c.App~=nil,
__dynamicKeys={TextColor3=b.Theme.Text.Primary[1]}},g'ImageLabel'{Name='Icon',
BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=c.AppIcon or'',
ImageColor3=b.Theme.Text.Primary[1].Value,Size=UDim2.fromOffset(20,20),Visible=c
.AppIcon~=nil,__dynamicKeys={ImageColor3=b.Theme.Text.Primary[1]}},g'UIPadding'{
Name='UIPadding',PaddingLeft=UDim.new(0,12),PaddingRight=UDim.new(0,12),
PaddingTop=UDim.new(0,12)}}},g'Folder'{Name='LayoutIgnore',g'TextButton'{Name=
'Exit',AnchorPoint=Vector2.new(0.5,0.5),AutoButtonColor=false,BackgroundColor3=b
.Theme.Controls.View[1].Value,BackgroundTransparency=0.1,BorderColor3=Color3.
fromRGB(0,0,0),BorderSizePixel=0,FontFace=Font.new'rbxassetid://12187365364',
Position=UDim2.fromOffset(3,3),Size=UDim2.fromOffset(20,20),Text='',TextColor3=b
.Theme.Text.Primary[1].Value,TextSize=14,TextTransparency=0.5,__dynamicKeys={
BackgroundColor3=b.Theme.Controls.View[1]},g'UICorner'{Name='UICorner',
CornerRadius=UDim.new(1,0)},g'ImageLabel'{Name='Icon',AnchorPoint=Vector2.new(
0.5,0.5),BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Image=
'rbxassetid://72660323302468',ImageColor3=b.Theme.Text.Primary[1].Value,
ImageTransparency=0.5,Position=UDim2.fromScale(0.5,0.5),Size=UDim2.fromOffset(20
,20),__dynamicKeys={ImageColor3=b.Theme.Text.Primary[1]}},g'Frame'{Name='Shadow'
,BackgroundColor3=Color3.fromRGB(255,255,255),BackgroundTransparency=1,
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Size=UDim2.fromScale(1,1),g
'UIStroke'{Name='UIStroke',ApplyStrokeMode=Enum.ApplyStrokeMode.Border,
Transparency=0.9},g'UICorner'{Name='UICorner',CornerRadius=UDim.new(1,0)}},g
'UIStroke'{Name='UIStroke',ApplyStrokeMode=Enum.ApplyStrokeMode.Border,Color=b.
Theme.Text.Primary[1].Value,Thickness=2,Transparency=0.96,__dynamicKeys={Color=b
.Theme.Text.Primary[1]}}}},g'UIScale'{Name='UIScale',Scale=0.85}})local j,k=i.
LayoutIgnore.Exit,i.Canvas local l,m=k.Content,k.Topbar return i,j,k,l,m end end
function a.M()local c=a.cache.M if not c then c={c=b()}a.cache.M=c end return c.
c end end do local b=function()a.b()return function(b,c)local d,e,f,g=a.e(),a.c(
),a.L(),a.M()local h=d.TweenService c=c or{}c.Title=c.Title or'Notification'c.
Subtitle=c.Subtitle or''c.Duration=c.Duration or 6 if c.App then c.App=string.
upper(c.App)end local i,j,k=false,4,{Age=0}local l,m,n,o,p=g(b,c,k,j)local q={
Title=function(q)o.TitleContainer.Title.Text=q end,Subtitle=function(q)o.
Subtitle.Text=q o.Subtitle.Visible=q~=''end,App=function(q)local r=string.upper(
q or'')p.App.Text=r p.App.Visible=q~=nil p.Visible=(q~=nil)or(c.AppIcon~=nil)if
c.App~=r then c.App=r end end,AppIcon=function(q)p.Icon.Image=q or''p.Icon.
Visible=q~=nil p.Visible=(c.App~=nil)or(q~=nil)end,Icon=function(q)if q then o.
TitleContainer.Icon.Image=q end o.TitleContainer.Icon.Visible=q~=nil end}local r
=e.Wrap(c,q,l,{'App','Icon'})r.Type='Notification'r.Theme=b.Theme e.Apply(c,r)
local s=function(s)if i then return end i=true if not l.Parent then return end
if c.Closed then task.spawn(c.Closed,r,s)end local t=h:Create(r.__instance,
TweenInfo.new(0.4,Enum.EasingStyle.Exponential,Enum.EasingDirection.Out),{
Position=r.__instance.Position+UDim2.fromOffset(350,0)})r:UpdateState(j,false)
local u=r.__instance:FindFirstChild'UIScale'if u then h:Create(u,TweenInfo.new(
0.4,Enum.EasingStyle.Exponential,Enum.EasingDirection.Out),{Scale=0.85}):Play()
end t:Play()t.Completed:Connect(function()if l.Parent then f.RemoveNotification(
r)l:Destroy()end end)end m.MouseButton1Click:Connect(function()s(true)end)
function r:Close(t)s(t)end function r:UpdateState(t,u)k.Age=t local v,w=t>=(j-1)
,t==0 local x,y,z,A=w and 0.1 or 1,w and 0.96 or 1,w and 0.9 or 1,w and 0.5 or 1
if v then x=1 y=1 z=1 A=1 end local B,C=v and 1 or 0.1,function(B,C)if u then
for D,E in pairs(C)do B[D]=E end else h:Create(B,TweenInfo.new(0.4,Enum.
EasingStyle.Exponential,Enum.EasingDirection.Out),C):Play()end end C(n,{
BackgroundTransparency=B})for D,E in ipairs(n:GetDescendants())do if E:IsA
'TextLabel'then local F=1 if not v then if E.Name=='Title'then F=self.Theme.Text
.Primary[2].Value elseif E.Name=='Subtitle'then F=self.Theme.Text.Secondary[2].
Value elseif E.Name=='App'then F=0.5 end end C(E,{TextTransparency=F})elseif E:
IsA'ImageLabel'and E.Name~='Icon'then C(E,{ImageTransparency=v and 1 or 0})end
end if o:FindFirstChild'TitleContainer'and o.TitleContainer:FindFirstChild'Icon'
then C(o.TitleContainer.Icon,{ImageTransparency=v and 1 or 0})end if p:
FindFirstChild'Icon'then C(p.Icon,{ImageTransparency=v and 1 or 0})end C(m,{
BackgroundTransparency=x})C(m.Icon,{ImageTransparency=A})C(m.UIStroke,{
Transparency=y})local D=m:FindFirstChild'Shadow'if D and D:FindFirstChild
'UIStroke'then C(D.UIStroke,{Transparency=z})end end l.Position=UDim2.new(1,
187.5,1,0)r:UpdateState(0,true)f.AddNotification(r,l)if l:FindFirstChild
'UIScale'then l.UIScale.Scale=0.85 h:Create(l.UIScale,TweenInfo.new(0.4,Enum.
EasingStyle.Exponential,Enum.EasingDirection.Out),{Scale=1}):Play()end h:Create(
l.__instance,TweenInfo.new(0.4,Enum.EasingStyle.Exponential,Enum.EasingDirection
.Out),{Position=UDim2.new(1,-162.5,1,0)}):Play()if c.Duration>0 then task.delay(
c.Duration,function()s()end)end return r end end function a.N()local c=a.cache.N
if not c then c={c=b()}a.cache.N=c end return c.c end end do local b=function()
return function(b)local c=a.d()local d,e=c.Create,b.__container or b.__instance
or b local f=(d'Frame'{Name='ImageSurface',BackgroundColor3=Color3.fromRGB(255,
255,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Size=UDim2.fromOffset(26,26),Parent=e,d'Frame'{Name='Surface',
AnchorPoint=Vector2.new(0.5,0.5),BackgroundColor3=Color3.fromRGB(200,200,200),
BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,Position=UDim2.fromScale(
0.5,0.5),Size=UDim2.fromOffset(24,24),d'ImageLabel'{Name='Image',AnchorPoint=
Vector2.new(0.5,0.5),BackgroundColor3=Color3.fromRGB(255,255,255),
BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),BorderSizePixel=0,
LayoutOrder=1,Position=UDim2.fromScale(0.5,0.5),Size=UDim2.fromOffset(20,20)},d
'UICorner'{Name='UICorner',CornerRadius=UDim.new(0,5)},d'UIGradient'{Name=
'UIGradient',Rotation=90,Enabled=true,__dynamicKeys={Color=b.Theme.Controls.
ImageSurface.Gradient}}}})local g={Body=f}g.Surface=(f:FindFirstChild'Surface')g
.Image=g.Surface and(g.Surface:FindFirstChild'Image')g.Gradient=g.Surface and(g.
Surface:FindFirstChild'UIGradient')return g end end function a.O()local c=a.
cache.O if not c then c={c=b()}a.cache.O=c end return c.c end end do local b=
function()a.b()return function(b,c)local d,e=a.c(),a.O()local f=e(b)c=c or{}c.
SurfaceColor=c.SurfaceColor or Color3.fromRGB(200,200,200)c.ImageColor=c.
ImageColor or Color3.fromRGB(255,255,255)c.Gradient=c.Gradient~=false local g,h=
{Image=function(g)f.Image.Image=g end,Gradient=function(g)f.Gradient.Enabled=g
end,SurfaceColor=function(g)f.Surface.BackgroundColor3=g end,ImageColor=function
(g)f.Image.ImageColor3=g end}h=d.Wrap(c,g,f.Body)h.Type='ImageSurface'h.Theme=b.
Theme h.Structures=f d.Apply(c,h)return h end end function a.P()local c=a.cache.
P if not c then c={c=b()}a.cache.P=c end return c.c end end do local b=function(
)local b,c,d,e=a.c(),a.f(),{Window=a.n(),Section=a.o(),Tab=a.q(),PageSection=a.
s(),Form=a.t(),Row=a.u(),HStack=a.v(),VStack=a.w(),TitleStack=a.x(),Label=a.y(),
Symbol=a.z(),Toggle=a.A(),TextField=a.B(),KeybindField=a.C(),Slider=a.D(),Button
=a.E(),Stepper=a.F(),RadioButtonGroup=a.G(),PopUpButton=a.J(),PullDownButton=a.
K(),Page=a.p(),Notification=a.N(),ImageSurface=a.P()},function(b)if debug and
debug.traceback then return debug.traceback(b)end return tostring(b)end local f=
function(f,g)return function(h,...)local i,j=table.pack(...),c.FromContext(h)
local k,l,m,n=j and j:_Begin(h,f,i[1]),xpcall(function()return g(h,table.unpack(
i,1,i.n))end,e)if not l then if j then j:_Cancel(k)end error(m,2)end if typeof(m
)=='Instance'or(typeof(m)=='table'and getmetatable(m)and typeof(getmetatable(m))
=='Instance')then n=m m=b.Wrap({},{},m)end if typeof(m)=='table'then if m.Type==
nil then m.Type=f end if m.Theme==nil and h and h.Theme then m.Theme=h.Theme end
end b.Apply(d,m)local o=n or(typeof(m)=='table'and pcall(function()return m.
__instance end)and m.__instance)if o~=nil then table.insert(__SH.COMPINDEXES,o)
if j then j:_Commit(k,m,o)end return m,o end if j then j:_Commit(k,m,nil)end
return m end end for g,h in pairs(d)do d[g]=f(g,h)end function d.register(g,h)d[
g]=f(g,h)end return d end function a.Q()local c=a.cache.Q if not c then c={c=b()
}a.cache.Q=c end return c.c end end do local b=function()local b=a.d()local c=b.
Value local d=function(d,e)local f=(typeof(d)=='Color3'and d)or(typeof(d)==
'string'and Color3.fromHex(d))return{c(f),c(1-(e/100))}end return{_id='Dark',
Text={Primary=d('FFFFFF',85),Secondary=d('FFFFFF',55),Tertiary=d('FFFFFF',25),
Quaternary=d('FFFFFF',10),Quinary=d('FFFFFF',5),SelectionPrimary=d('FFFFFF',100)
,PrimaryAccent=d('FFFFFF',38)},Accents={Red=d('FF453A',100)},Controls={
Background=d('1C1C1E',100),View=d('1F1F21',100),ViewBorder=d('FFFFFF',5),
WindowControlDisabled=d('FFFFFF',20),WindowControlIcon=d('000000',50),
WindowControlStroke=d('FFFFFF',10),Exit=d('FF5F57',100),Minimize=d('FEBC2E',100)
,Zoom=d('28C840',100),SwitchAccent=d('478CF6',100),Selection=d('007AFF',100),
SelectionStroke=d('007AFF',60),SelectionFocused=d('0A82FF',100),
SelectionFocusedAccent=d('FFFFFF',85),Sidebar=d('202023',84),ScrollBar=d(
'FFFFFF',35),Separator={Background=d('000000',50),Shadow=d('FFFFFF',0)},Titlebar
=d('363636',100),TitlebarShadow={Background=d('000000',0),Color=c(ColorSequence.
new{ColorSequenceKeypoint.new(0,Color3.fromRGB(0,0,0)),ColorSequenceKeypoint.
new(1,Color3.fromRGB(255,255,255))}),Transparency=c(NumberSequence.new{
NumberSequenceKeypoint.new(0,0.5),NumberSequenceKeypoint.new(1,1)})},Toggle={
Knob=d('FFFFFF',100),KnobEffects=d('FFFFFF',100),SwitchOff=d('7a7a7a',40),
SwitchOn=d('478cf6',100),DepthEffect=c(ColorSequence.new{ColorSequenceKeypoint.
new(0,Color3.fromRGB(225,225,225)),ColorSequenceKeypoint.new(0.68,Color3.
fromRGB(255,255,255)),ColorSequenceKeypoint.new(1,Color3.fromRGB(255,255,255))})
},Slider={Track=d('2C2C2E',100),TrackEffects=d('000000',10),Thumb=d('FFFFFF',100
),ThumbStroke=d('000000',20),ThumbEffects=d('FFFFFF',80)},Button={Shadow=c(
Color3.fromRGB(0,0,0)),FillPrimary=c(ColorSequence.new{ColorSequenceKeypoint.
new(0,Color3.fromRGB(72,148,255)),ColorSequenceKeypoint.new(1,Color3.fromRGB(10,
110,255))}),FillSecondary=c(ColorSequence.new{ColorSequenceKeypoint.new(0,Color3
.fromRGB(60,60,60)),ColorSequenceKeypoint.new(1,Color3.fromRGB(55,55,55))})},
Stepper={Background=d('373737',100),Dropshadow=d('000000',100),Separator=d(
'FFFFFF',10),Filler=d('FFFFFF',4),SegmentShadow=c(ColorSequence.new{
ColorSequenceKeypoint.new(0,Color3.fromRGB(0,0,0)),ColorSequenceKeypoint.new(1,
Color3.fromRGB(0,0,0))})},RadioButtonGroup={Background=d('373737',100),Dot=d(
'FFFFFF',100),Stroke=d('000000',20),Overlay=d('FFFFFF',8),InnerShadow=d('FFFFFF'
,10)},MenuButton={IndicatorBackground=d('FFFFFF',10),MenuBackground=d('2C2C2E',
95)},ImageSurface={Gradient=c(ColorSequence.new{ColorSequenceKeypoint.new(0,
Color3.fromRGB(255,255,255)),ColorSequenceKeypoint.new(1,Color3.fromRGB(190,190,
190))})}}}end function a.R()local c=a.cache.R if not c then c={c=b()}a.cache.R=c
end return c.c end end do local b=function()local b=a.d()local c=b.Value local d
=function(d,e)local f=(typeof(d)=='Color3'and d)or(typeof(d)=='string'and Color3
.fromHex(d))return{c(f),c(1-(e/100))}end return{_id='Light',Text={Primary=d(
'000000',85),Secondary=d('000000',50),Tertiary=d('000000',25),Quaternary=d(
'000000',10),Quinary=d('000000',5),SelectionPrimary=d('FFFFFF',100),
PrimaryAccent=d('4D4D4D',100)},Accents={Red=d('FF3B30',100)},Controls={
Background=d('FFFFFF',100),View=d('FCFCFC',100),ViewBorder=d('000000',5),
WindowControlDisabled=d('000000',20),WindowControlIcon=d('000000',50),
WindowControlStroke=d('000000',20),Exit=d('FF5F57',100),Minimize=d('FEBC2E',100)
,Zoom=d('28C840',100),SwitchAccent=d('478CF6',100),Selection=d('007AFF',100),
SelectionStroke=d('007AFF',50),SelectionFocused=d('0A82FF',100),
SelectionFocusedAccent=d('FFFFFF',85),Sidebar=d('EAEAEA',84),ScrollBar=d(
'000000',40),Separator={Background=d('000000',18),Shadow=d('000000',10)},
Titlebar=d('EEEEEE',100),TitlebarShadow={Background=d('EAEAEA',25),Color=c(
ColorSequence.new{ColorSequenceKeypoint.new(0,Color3.fromRGB(255,255,255)),
ColorSequenceKeypoint.new(1,Color3.fromRGB(0,0,0))}),Transparency=c(
NumberSequence.new{NumberSequenceKeypoint.new(0,0.35),NumberSequenceKeypoint.
new(1,0.35)})},Toggle={Knob=d('FFFFFF',100),KnobEffects=d('FFFFFF',100),
SwitchOff=d('000000',9),SwitchOn=d('478CF6',100),DepthEffect=c(ColorSequence.new
{ColorSequenceKeypoint.new(0,Color3.fromRGB(225,225,225)),ColorSequenceKeypoint.
new(0.68,Color3.fromRGB(255,255,255)),ColorSequenceKeypoint.new(1,Color3.
fromRGB(255,255,255))})},Slider={Track=d('000000',5),TrackEffects=d('000000',0),
Thumb=d('FFFFFF',100),ThumbStroke=d('000000',2),ThumbEffects=d('FFFFFF',100)},
Button={Shadow=c(Color3.new(0,0,0)),FillPrimary=c(ColorSequence.new{
ColorSequenceKeypoint.new(0,Color3.fromRGB(43,145,255)),ColorSequenceKeypoint.
new(1,Color3.fromRGB(0,122,255))}),FillSecondary=c(ColorSequence.new{
ColorSequenceKeypoint.new(0,Color3.fromRGB(255,255,255)),ColorSequenceKeypoint.
new(1,Color3.fromRGB(255,255,255))})},Stepper={Background=d('FFFFFF',100),
Dropshadow=d('000000',100),Separator=d('000000',22),Filler=d('000000',5),
SegmentShadow=c(ColorSequence.new{ColorSequenceKeypoint.new(0,Color3.fromRGB(0,0
,0)),ColorSequenceKeypoint.new(1,Color3.fromRGB(255,255,255))})},
RadioButtonGroup={Background=d('FFFFFF',100),Dot=d('FFFFFF',100),Stroke=d(
'000000',8),Overlay=d('FFFFFF',17),InnerShadow=d('000000',10)},MenuButton={
IndicatorBackground=d('000000',5),MenuBackground=d('F6F6F6',95)},ImageSurface={
Gradient=c(ColorSequence.new{ColorSequenceKeypoint.new(0,Color3.fromRGB(255,255,
255)),ColorSequenceKeypoint.new(1,Color3.fromRGB(190,190,190))})}}}end function
a.S()local c=a.cache.S if not c then c={c=b()}a.cache.S=c end return c.c end end
do local b=function()return{Dark=a.R(),Light=a.S()}end function a.T()local c=a.
cache.T if not c then c={c=b()}a.cache.T=c end return c.c end end do local b=
function()local b=function(b,c)return ColorSequence.new{ColorSequenceKeypoint.
new(0,Color3.fromHex(b)),ColorSequenceKeypoint.new(1,Color3.fromHex(c))}end
return{Blue={_id='Blue',Dark={SwitchAccent=Color3.fromHex'#0A84FF',Selection=
Color3.fromHex'#007AFF',SelectionFocused=Color3.fromHex'#0A82FF',Toggle={
SwitchOn=Color3.fromHex'#0A84FF'},Button={FillPrimary=ColorSequence.new{
ColorSequenceKeypoint.new(0,Color3.fromRGB(72,148,255)),ColorSequenceKeypoint.
new(1,Color3.fromRGB(10,110,255))}}},Light={SwitchAccent=Color3.fromHex'#0A84FF'
,Selection=Color3.fromHex'#007AFF',SelectionFocused=Color3.fromHex'#0A82FF',
Toggle={SwitchOn=Color3.fromHex'#0A84FF'},Button={FillPrimary=ColorSequence.new{
ColorSequenceKeypoint.new(0,Color3.fromRGB(43,145,255)),ColorSequenceKeypoint.
new(1,Color3.fromRGB(0,122,255))}}}},Purple={_id='Purple',Dark={SwitchAccent=
Color3.fromHex'#BF5AF2',Selection=Color3.fromHex'#9B35D4',SelectionFocused=
Color3.fromHex'#AE4FE0',Toggle={SwitchOn=Color3.fromHex'#BF5AF2'},Button={
FillPrimary=b('#AE4FE0','#7A18B8')}},Light={SwitchAccent=Color3.fromHex'#AF52DE'
,Selection=Color3.fromHex'#8C2EC0',SelectionFocused=Color3.fromHex'#9E42CA',
Toggle={SwitchOn=Color3.fromHex'#AF52DE'},Button={FillPrimary=b('#9E42CA',
'#7010A8')}}},Pink={_id='Pink',Dark={SwitchAccent=Color3.fromHex'#FF375F',
Selection=Color3.fromHex'#D4163E',SelectionFocused=Color3.fromHex'#E83058',
Toggle={SwitchOn=Color3.fromHex'#FF375F'},Button={FillPrimary=b('#E83058',
'#C00030')}},Light={SwitchAccent=Color3.fromHex'#FF2D55',Selection=Color3.
fromHex'#C8103A',SelectionFocused=Color3.fromHex'#DC2A50',Toggle={SwitchOn=
Color3.fromHex'#FF2D55'},Button={FillPrimary=b('#DC2A50','#B4002C')}}},Red={_id=
'Red',Dark={SwitchAccent=Color3.fromHex'#FF453A',Selection=Color3.fromHex
'#D4201A',SelectionFocused=Color3.fromHex'#E83830',Toggle={SwitchOn=Color3.
fromHex'#FF453A'},Button={FillPrimary=b('#E83830','#BC0E08')}},Light={
SwitchAccent=Color3.fromHex'#FF3B30',Selection=Color3.fromHex'#C81A10',
SelectionFocused=Color3.fromHex'#DC2E24',Toggle={SwitchOn=Color3.fromHex
'#FF3B30'},Button={FillPrimary=b('#DC2E24','#B00808')}}},Orange={_id='Orange',
Dark={SwitchAccent=Color3.fromHex'#FF9F0A',Selection=Color3.fromHex'#C86800',
SelectionFocused=Color3.fromHex'#E07C00',Toggle={SwitchOn=Color3.fromHex
'#FF9F0A'},Button={FillPrimary=b('#E07C00','#A85000')}},Light={SwitchAccent=
Color3.fromHex'#FF9500',Selection=Color3.fromHex'#BC6000',SelectionFocused=
Color3.fromHex'#D47000',Toggle={SwitchOn=Color3.fromHex'#FF9500'},Button={
FillPrimary=b('#D47000','#A04800')}}},Yellow={_id='Yellow',Dark={SwitchAccent=
Color3.fromHex'#FFD60A',Selection=Color3.fromHex'#A87800',SelectionFocused=
Color3.fromHex'#C49000',Toggle={SwitchOn=Color3.fromHex'#FFD60A'},Button={
FillPrimary=b('#C49000','#8C6000')}},Light={SwitchAccent=Color3.fromHex'#FFCC00'
,Selection=Color3.fromHex'#9C7000',SelectionFocused=Color3.fromHex'#B88400',
Toggle={SwitchOn=Color3.fromHex'#FFCC00'},Button={FillPrimary=b('#B88400',
'#845800')}}},Green={_id='Green',Dark={SwitchAccent=Color3.fromHex'#32D74B',
Selection=Color3.fromHex'#1A9030',SelectionFocused=Color3.fromHex'#24A83C',
Toggle={SwitchOn=Color3.fromHex'#32D74B'},Button={FillPrimary=b('#24A83C',
'#0E7020')}},Light={SwitchAccent=Color3.fromHex'#28CD41',Selection=Color3.
fromHex'#148428',SelectionFocused=Color3.fromHex'#1E9C34',Toggle={SwitchOn=
Color3.fromHex'#28CD41'},Button={FillPrimary=b('#1E9C34','#0C6818')}}},Graphite=
{_id='Graphite',Dark={SwitchAccent=Color3.fromHex'#98989D',Selection=Color3.
fromHex'#58585C',SelectionFocused=Color3.fromHex'#6E6E72',Toggle={SwitchOn=
Color3.fromHex'#98989D'},Button={FillPrimary=b('#6E6E72','#404044')}},Light={
SwitchAccent=Color3.fromHex'#8E8E93',Selection=Color3.fromHex'#4E4E52',
SelectionFocused=Color3.fromHex'#626266',Toggle={SwitchOn=Color3.fromHex
'#8E8E93'},Button={FillPrimary=b('#626266','#383838')}}}}end function a.U()local
c=a.cache.U if not c then c={c=b()}a.cache.U=c end return c.c end end end local
b,c,d,e,f,g,h,i,j,k=a.a(),a.b(),a.d(),a.c(),a.e(),a.f(),a.g(),a.Q(),a.T(),a.U()
local l,m,n,o,p=d.Create,f.TweenService,{Themes=j,Accents=k,Symbols=h,Creator=d,
Binder=e,Components=i,AppRecorder=g},(TweenInfo.new(0.4,Enum.EasingStyle.
Exponential,Enum.EasingDirection.Out))local function q(r,s)local t={}for u,v in
pairs(r)do local w=typeof(v)if w=='table'then if v.Value~=nil and v.Connect and
typeof(v.Connect)=='function'then t[u]=d.Value(v.Value)else t[u]=q(v)end else t[
u]=v end end if s and not r._id then r._id=s end return t end local function r(s
,t)for u,v in pairs(t)do local w=s[u]if not w and type(s)=='table'and type(s.
Controls)=='table'then w=s.Controls[u]end if not w then continue end if type(w)
=='table'and w.Connect then w.Value=v elseif type(w)=='table'and w[1]and w[1].
Connect and typeof(v)=='Color3'then w[1].Value=v elseif type(w)=='table'and
type(v)=='table'then r(w,v)end end end local s=function(s,t,u)local function v(w
,x)for y,z in pairs(x)do if type(z)=='table'and type(w[y])=='table'and not z.
Value then v(w[y],z)elseif w[y]and z and z.Value~=nil then w[y].Value=z.Value
end end end v(s,t)if u and u[t._id]then r(s,u[t._id])end end n.New=function(t)if
not game:IsLoaded()then game.Loaded:Wait()end t=t or{}local u,v=t.Theme or j.
Light,t.Accent or k.Blue local w=u t.Theme=q(u)t.Accent=q(v)s(t.Theme,u,v)local
x=(b.ProtectUI(l'ScreenGui'{Name='Cascade',IgnoreGuiInset=true,ResetOnSpawn=
false,ZIndexBehavior=Enum.ZIndexBehavior.Sibling,DisplayOrder=200,
OnTopOfCoreBlur=true}))local y=(l'ImageButton'{Name='WindowPill',AnchorPoint=
Vector2.new(0.5,0),AutoButtonColor=false,BackgroundColor3=Color3.fromRGB(255,255
,255),BackgroundTransparency=1,BorderColor3=Color3.fromRGB(0,0,0),
BorderSizePixel=0,Image='rbxassetid://93520763686656',ImageTransparency=0.5,
Position=UDim2.new(0.5,0,0,10),Size=UDim2.fromOffset(180,5),Parent=x.__instance,
l'UICorner'{Name='UICorner',CornerRadius=UDim.new(1,0)}})y.MouseEnter:Connect(
function()if p then p:Cancel()end p=m:Create(y.__instance,o,{ImageTransparency=
0.15})if p then p:Play()end end)y.MouseLeave:Connect(function()if p then p:
Cancel()end p=m:Create(y.__instance,o,{ImageTransparency=0.5})if p then p:Play()
end end)local z=e.Wrap(t,{WindowPill=function(z)y.Visible=z end,Theme=function(z
)w=z s(t.Theme,z,t.Accent)end,Accent=function(z)t.Accent=q(z,z._id)s(t.Theme,w,z
)end},x,{'Theme','Accent'})z.Structures={WindowPill=y}setmetatable(t,{__index=i}
)e.Apply(t,z)task.defer(e.Apply,t,z)return z end n.Component=function(t)t=t or{}
local u,v=t.Theme or j.Light,t.Accent or k.Blue local w=u t.Theme=q(u)t.Accent=
q(v)s(t.Theme,u,v)local x=e.Wrap(t,{Theme=function(x)w=x s(t.Theme,x,t.Accent)
end,Accent=function(x)t.Accent=q(x,x._id)s(t.Theme,w,x)end},nil,{'Theme',
'Accent'})if t.Parent then x.__container=t.Parent end setmetatable(t,{__index=i}
)e.Apply(t,x)task.defer(e.Apply,t,x)return x end n.RegisterComponent=function(t,
u)i.register(t,u)end n.AppDump=function(t,u)return g.DumpApp(t,u)end return n

]====], "CascadeMerged")()
assert(cascade, "Cascade failed to load")

-- Source-1 feature dependencies first.
local Backend = runEmbed(EMBED.SosyBackend, "SosyBackend")
_G.SosyBackend = Backend

pcall(function()
    local pack = runEmbed(EMBED.ShaderPack, "ShaderPack")
    if type(pack) == "table" then _G._SosyShaderPack = pack end
end)

pcall(function()
    local shaders = runEmbed(EMBED.SosyShaders, "SosyShaders")
    _G.SosyShaders = shaders
end)

pcall(function()
    runEmbed(EMBED.NovaCore, "NovaCore")
    _G._SosyNovaCoreLoaded = true
end)

pcall(function()
    if type(Backend.bootstrapFromVps) == "function" then
        Backend.bootstrapFromVps()
    end
end)

-- Cascade -> FeatureMount compatibility bridge.
-- It deliberately exposes the same control surface expected by source 1,
-- while rendering everything through Cascade from source 2.
local function makeCascadeBridge(lib)
    local W = {}

    function W:Window(cfg)
        cfg = cfg or {}
        local app = lib.New({
            Theme = lib.Themes.Dark,
            Accent = lib.Accents.Blue,
        })
        local window = app:Window({
            Title = tostring(cfg.Title or "SosyHUB"),
            Subtitle = tostring(cfg.Subtitle or "Bastion"),
        })

        local menu = window:Section({
            Title = "Menu",
            Disclosure = false,
        })

        local wrapper = {
            root = window,
            _app = app,
            _elementRegistry = {},
            _toggleRegistry = {},
        }

        local tabs = {}

        function wrapper:Tab(tc)
            tc = tc or {}
            local name = tostring(tc.Name or tc.name or "Tab")
            if tabs[name] then return tabs[name] end

            local tabObj = menu:Tab({
                Title = name,
                Selected = next(tabs) == nil,
            })

            local sections = {}
            local T = {}

            function T:Section(sp)
                sp = sp or {}
                local secName = tostring(sp.Name or sp.name or "Main")
                local cacheKey = secName .. "|" .. tostring(sp.side or "")
                if sections[cacheKey] then return sections[cacheKey] end

                local form = tabObj:PageSection({
                    Title = secName,
                }):Form()

                local S = {}

                function S:Toggle(p)
                    p = p or {}
                    local row = form:Row()
                    row:Left():TitleStack({ Title = tostring(p.Name or p.name or "") })

                    local ctl = row:Right():Toggle({
                        Value = p.Default == true or p.default == true,
                        ValueChanged = function(_, v)
                            if type(p.Callback) == "function" then
                                p.Callback(v)
                            end
                        end,
                    })

                    local api = {}
                    function api:set(v, fire)
                        if fire == false then
                            -- Cascade's ValueChanged is event driven. The backend
                            -- already owns state, so this is intentionally direct.
                            pcall(function() ctl.Value = v == true end)
                        else
                            pcall(function() ctl.Value = v == true end)
                        end
                    end
                    function api:Set(v) api:set(v, true) end
                    function api:Get()
                        local ok, value = pcall(function() return ctl.Value end)
                        return ok and value or false
                    end

                    wrapper._elementRegistry[p.Name or p.name or "Toggle"] = api
                    table.insert(wrapper._toggleRegistry, { name = p.Name or p.name, api = api })
                    return api
                end

                function S:Slider(p)
                    p = p or {}
                    local minV = tonumber(p.Minimum or p.min) or 0
                    local maxV = tonumber(p.Maximum or p.max) or 100
                    local defaultV = tonumber(p.Default or p.default) or minV

                    local row = form:Row()
                    row:Left():TitleStack({ Title = tostring(p.Name or p.name or "") })

                    local ctl = row:Right():Slider({
                        Minimum = minV,
                        Maximum = maxV,
                        Value = defaultV,
                        ValueChanged = function(_, v)
                            if type(p.Callback) == "function" then
                                p.Callback(v)
                            end
                        end,
                    })

                    local api = {}
                    function api:Set(v)
                        pcall(function() ctl.Value = tonumber(v) or minV end)
                    end
                    function api:set(v)
                        api:Set(v)
                    end
                    function api:Get()
                        local ok, value = pcall(function() return ctl.Value end)
                        return ok and value or defaultV
                    end
                    wrapper._elementRegistry[p.Name or p.name or "Slider"] = api
                    return api
                end

                local function makeDropdown(p)
                    p = p or {}
                    local opts = p.Options or p.items or {}
                    local current = p.Default or p.default or opts[1]

                    local row = form:Row()
                    row:Left():TitleStack({ Title = tostring(p.Name or p.name or "") })

                    local ctl = row:Right():PullDownButton({
                        Label = tostring(p.Name or p.name or "Select"),
                        Options = opts,
                        Value = current,
                        ValueChanged = function(self2, v)
                            if type(p.Callback) == "function" then
                                p.Callback(self2.Options[v] or v)
                            end
                        end,
                    })

                    local api = {}

                    function api:ClearOptions()
                        pcall(function()
                            while #ctl.Options > 0 do ctl:Remove(1) end
                        end)
                    end

                    function api:InsertOptions(list)
                        pcall(function()
                            for _, v in ipairs(list or {}) do
                                ctl:Option(tostring(v))
                            end
                        end)
                    end

                    function api:set_items(list)
                        api:ClearOptions()
                        api:InsertOptions(list or {})
                    end

                    function api:SetItems(list) api:set_items(list) end
                    function api:SetOptions(list) api:set_items(list) end
                    function api:SetList(list) api:set_items(list) end

                    function api:set_value(v)
                        pcall(function()
                            ctl.Value = v
                        end)
                    end
                    function api:SetValue(v) api:set_value(v) end
                    function api:SetSelected(v) api:set_value(v) end
                    function api:Select(v) api:set_value(v) end
                    function api:Get()
                        local ok, value = pcall(function() return ctl.Value end)
                        return ok and value or current
                    end

                    wrapper._elementRegistry[p.Name or p.name or "Dropdown"] = api
                    return api
                end

                function S:Dropdown(p)
                    return makeDropdown(p)
                end

                function S:List(p)
                    return makeDropdown(p)
                end

                function S:Textbox(p)
                    p = p or {}
                    local row = form:Row()
                    row:Left():TitleStack({ Title = tostring(p.Name or p.name or "") })

                    local ctl = row:Right():TextField({
                        Placeholder = tostring(p.Placeholder or p.placeholder or ""),
                        Value = tostring(p.Default or p.default or ""),
                        TextChanged = function(_, v)
                            if type(p.Callback) == "function" then
                                p.Callback(v)
                            end
                        end,
                    })

                    local api = {}
                    function api:Set(v)
                        pcall(function() ctl.Value = tostring(v or "") end)
                    end
                    function api:set(v) api:Set(v) end
                    function api:Get()
                        local ok, value = pcall(function() return ctl.Value end)
                        return ok and value or ""
                    end
                    wrapper._elementRegistry[p.Name or p.name or "Textbox"] = api
                    return api
                end

                function S:Button(p)
                    p = p or {}
                    local row = form:Row()
                    row:Right():Button({
                        Label = tostring(p.Name or p.name or "Button"),
                        State = false,
                        BindPressed = function()
                            if type(p.Callback) == "function" then
                                p.Callback()
                            end
                        end,
                    })
                    return {}
                end

                function S:Label(p)
                    p = p or {}
                    local row = form:Row()
                    local label = row:Left():Label({
                        Text = tostring(p.Name or p.name or p.Text or p.text or ""),
                    })
                    local api = {}
                    function api:set(v)
                        pcall(function() label.Text = tostring(v or "") end)
                    end
                    function api:Set(v) api:set(v) end
                    wrapper._elementRegistry[p.Name or p.name or "Label"] = api
                    return api
                end

                sections[cacheKey] = S
                return S
            end

            tabs[name] = T
            return T
        end

        return wrapper
    end

    return W
end

local Lib = makeCascadeBridge(cascade)
local win = Lib:Window({
    Title = "SosyHUB",
    Subtitle = "Bastion • Cascade",
})

-- Mount every source-1 catalog entry and its original Backend callbacks.
local mountFn = runEmbed(EMBED.FeatureMountStandalone, "FeatureMountStandalone")
assert(type(mountFn) == "function", "FeatureMountStandalone did not return a function")

local mountOk, mountErr = pcall(function()
    mountFn(win, Lib)
end)

if not mountOk then
    warn("[SosyHUB] Feature mount failed: " .. tostring(mountErr))
else
    warn("[SosyHUB] ALL FEATURES MOUNTED THROUGH CASCADE")
end

-- Keep dynamic lists from source 1 alive.
task.defer(function()
    task.wait(0.25)

    pcall(function()
        if not _G.SosyBackend then return end
        local items = {"None"}
        local folder = workspace:FindFirstChild("Items")
        if folder then
            local seen = {}
            for _, obj in ipairs(folder:GetDescendants()) do
                if (obj:IsA("Tool") or obj:IsA("Model") or obj:IsA("BasePart"))
                    and obj.Name ~= "" and not seen[obj.Name] then
                    seen[obj.Name] = true
                    table.insert(items, obj.Name)
                end
            end
        end
        table.sort(items, function(a, b)
            return a ~= "None" and (b == "None" or a:lower() < b:lower())
        end)
        _G.SosyBackend.setOptions("Select Item", items)
    end)

    pcall(function()
        if type(_G.pianoScanSongs) ~= "function" or not _G.SosyBackend then return end
        local songs = _G.pianoScanSongs() or {}
        local opts = {"None"}
        for _, sng in ipairs(songs) do
            table.insert(opts, tostring(sng))
        end
        table.sort(opts, function(a, b)
            return a ~= "None" and (b == "None" or a:lower() < b:lower())
        end)
        _G.SosyBackend.setOptions("Song", opts)
    end)
end)

-- Compatibility exports retained for scripts that query the old hub.
_G._SosyNovaWin = win
_G._SosyNovaHubBuilt = true
_G._SosyNovaFeaturesMounted = mountOk
_G.SosyNovaOpenHub = function()
    return win
end

warn("[SosyHUB] Cascade merge ready.")
