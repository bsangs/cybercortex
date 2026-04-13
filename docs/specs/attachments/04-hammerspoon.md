# 04 — Hammerspoon Scripts

## agent/hammerspoon/watcher.lua

```lua
-- watcher.lua
-- Tracks app switching, browser URLs, idle state, and system events.
-- Logs everything to a daily activity file in sources/raw/.

local M  = {}

local BASE_DIR       = os.getenv("HOME") .. "/{project}"
local RAW_DIR        = BASE_DIR .. "/sources/raw"
local IDLE_THRESHOLD = 600  -- seconds (10 min)

local _appWatcher      = nil
local _caff            = nil
local _idleTap         = nil
local _idleTimer       = nil
local _lastActivityTime = os.time()
local _idleFired       = false

local function ensureDir(path)
    hs.execute("mkdir -p '" .. path:gsub("'", "'\\''") .. "'")
end

local function today()
    return os.date("%Y-%m-%d")
end

local function timeNow()
    return os.date("%H:%M:%S")
end

local function activityFilePath()
    return RAW_DIR .. "/activity-" .. today() .. ".md"
end

local function ensureHeader(path)
    local f = io.open(path, "r")
    if f then
        local content = f:read("*a")
        f:close()
        if content and #content > 0 then return end
    end
    local fw, err = io.open(path, "w")
    if not fw then
        print("[watcher] ERROR: cannot create activity file: " .. tostring(err))
        return
    end
    fw:write("---\n")
    fw:write("source: app-watcher\n")
    fw:write("date: " .. today() .. "\n")
    fw:write("---\n\n")
    fw:close()
end

local function log(line)
    local path = activityFilePath()
    ensureDir(RAW_DIR)
    ensureHeader(path)
    local f, err = io.open(path, "a")
    if not f then
        print("[watcher] ERROR: cannot open activity file: " .. tostring(err))
        return
    end
    f:write("- " .. timeNow() .. " " .. line .. "\n")
    f:close()
    print("[watcher] " .. line)
end

local function onAppActivated(appName, _eventType, _app)
    if _eventType ~= hs.application.watcher.activated then return end
    local win   = hs.window.focusedWindow()
    local title = win and win:title() or ""
    if title ~= "" then
        log(string.format("[focus] %s — %s", appName, title))
    else
        log(string.format("[focus] %s", appName))
    end
end

function M.triggerSessionEndIngest()
    local result = hs.execute("ls -1 '" .. RAW_DIR .. "' 2>/dev/null | wc -l")
    local count  = tonumber(result and result:match("%d+")) or 0
    if count == 0 then
        print("[watcher] triggerSessionEndIngest: no files, skipping")
        return
    end
    print(string.format("[watcher] triggerSessionEndIngest: %d file(s), triggering Claude ingest", count))
    local task = hs.task.new(
        "/opt/homebrew/bin/claude",
        function(exitCode, stdout, stderr)
            print(string.format("[watcher] ingest done (exit=%d)", exitCode))
        end,
        function(task, stdout, stderr) return true end,
        { "--print", "--dangerously-skip-permissions", "ingest sources/raw/ into memory" }
    )
    task:setWorkingDirectory(BASE_DIR)
    task:start()
end

local function onCaffeinateEvent(eventType)
    local E = hs.caffeinate.watcher
    if eventType == E.screensDidLock then
        log("[lock] 화면잠금")
        M.triggerSessionEndIngest()
    elseif eventType == E.screensDidUnlock then
        log("[wake] 잠금해제")
    elseif eventType == E.screensDidSleep then
        log("[lid] 닫힘")
        M.triggerSessionEndIngest()
    elseif eventType == E.screensDidWake then
        log("[lid] 열림")
    elseif eventType == E.systemWillSleep then
        log("[sleep] 시스템 슬립")
    elseif eventType == E.systemDidWake then
        log("[wake] 시스템 깨어남")
    end
end

local function resetIdle()
    _lastActivityTime = os.time()
    _idleFired        = false
end

local function checkIdle()
    local elapsed = os.time() - _lastActivityTime
    if elapsed >= IDLE_THRESHOLD and not _idleFired then
        _idleFired = true
        log("[idle] 10분 무입력")
    end
end

function M.start()
    if _appWatcher then _appWatcher:stop() end
    _appWatcher = hs.application.watcher.new(onAppActivated)
    _appWatcher:start()

    if _caff then _caff:stop() end
    _caff = hs.caffeinate.watcher.new(onCaffeinateEvent)
    _caff:start()

    if _idleTap then _idleTap:stop() end
    _idleTap = hs.eventtap.new(
        { hs.eventtap.event.types.keyDown, hs.eventtap.event.types.leftMouseDown },
        function(_event) resetIdle(); return false end
    )
    _idleTap:start()

    if _idleTimer then _idleTimer:stop() end
    _idleTimer = hs.timer.new(60, checkIdle)
    _idleTimer:start()
    resetIdle()
    print("[watcher] All watchers started")
end

function M.stop()
    if _appWatcher then _appWatcher:stop(); _appWatcher = nil end
    if _caff then _caff:stop(); _caff = nil end
    if _idleTap then _idleTap:stop(); _idleTap = nil end
    if _idleTimer then _idleTimer:stop(); _idleTimer = nil end
    print("[watcher] All watchers stopped")
end

return M
```

## agent/hammerspoon/kakaotalk.lua

```lua
-- kakaotalk.lua
-- Launches external Swift binary to read KakaoTalk via AX API.

local M = {}

local BASE_DIR = os.getenv("HOME") .. "/{project}"
local READER_BIN = BASE_DIR .. "/agent/scripts/kakaotalk-reader"

local _timer = nil
local _running = false

function M.collect()
    if _running then
        print("[kakaotalk] Already running, skipping")
        return
    end
    _running = true
    hs.task.new(READER_BIN, function(exitCode, stdout, stderr)
        _running = false
        if stdout and stdout ~= "" then
            for line in stdout:gmatch("[^\n]+") do print(line) end
        end
        if stderr and stderr ~= "" then
            print("[kakaotalk] stderr: " .. stderr)
        end
    end):start()
end

function M.startTimer(interval)
    interval = interval or 30
    if _timer then _timer:stop(); _timer = nil end
    print(string.format("[kakaotalk] Starting timer, interval=%ds", interval))
    M.collect()
    _timer = hs.timer.new(interval, function() M.collect() end)
    _timer:start()
end

function M.stopTimer()
    if _timer then _timer:stop(); _timer = nil; print("[kakaotalk] Timer stopped")
    else print("[kakaotalk] No timer running") end
end

return M
```

## agent/hammerspoon/ax-dump.lua

```lua
-- ax-dump.lua
-- AX Tree Explorer for structure discovery

local ax = require("hs.axuielement")
local M = {}

local BASE_DIR = os.getenv("HOME") .. "/{project}"
local RAW_DIR = BASE_DIR .. "/sources/raw"

local function sanitize(s)
    if not s then return nil end
    return tostring(s)
end

local function formatElement(elem, depth)
    local indent = string.rep("  ", depth)
    local role        = elem:attributeValue("AXRole")        or ""
    local subrole     = elem:attributeValue("AXSubrole")
    local title       = elem:attributeValue("AXTitle")
    local value       = elem:attributeValue("AXValue")
    local description = elem:attributeValue("AXDescription")
    local identifier  = elem:attributeValue("AXIdentifier")
    title       = sanitize(title)
    description = sanitize(description)
    if value ~= nil then value = sanitize(tostring(value)) end
    local parts = { "[" .. role }
    if subrole and subrole ~= "" then parts[#parts + 1] = " sub=" .. subrole end
    if identifier and identifier ~= "" then parts[#parts + 1] = " id=" .. identifier end
    if title and title ~= "" then parts[#parts + 1] = " title=" .. title end
    if description and description ~= "" then parts[#parts + 1] = " desc=" .. description end
    if value and value ~= "" then parts[#parts + 1] = " val=" .. value end
    parts[#parts + 1] = "]"
    return indent .. table.concat(parts)
end

local function traverse(elem, depth, maxDepth, lines)
    if depth > maxDepth then return end
    if elem == nil then return end
    local ok, line = pcall(formatElement, elem, depth)
    if ok then lines[#lines + 1] = line
    else lines[#lines + 1] = string.rep("  ", depth) .. "[ERROR: " .. tostring(line) .. "]" end
    if depth < maxDepth then
        local ok2, children = pcall(function() return elem:attributeValue("AXChildren") or {} end)
        if ok2 and children then
            for _, child in ipairs(children) do traverse(child, depth + 1, maxDepth, lines) end
        end
    end
end

local function timestamp() return os.date("%Y%m%d-%H%M%S") end

local function writeFile(path, lines)
    local f, err = io.open(path, "w")
    if not f then print("[ax-dump] ERROR: " .. tostring(err)); return end
    for _, line in ipairs(lines) do f:write(line .. "\n") end
    f:close()
    print("[ax-dump] Saved: " .. path .. " (" .. #lines .. " lines)")
end

local function getAppElement(appName)
    local app = hs.application.find(appName)
    if not app then print("[ax-dump] ERROR: app not found: " .. appName); return nil end
    return ax.applicationElement(app)
end

function M.dump(appName, maxDepth)
    appName = appName or "Finder"; maxDepth = maxDepth or 10
    local appElem = getAppElement(appName)
    if not appElem then return end
    local ts = timestamp()
    local outPath = RAW_DIR .. "/ax-dump-" .. appName .. "-" .. ts .. ".txt"
    local lines = { "=== AX Dump ===", "App: " .. appName, "Date: " .. os.date("%Y-%m-%d %H:%M:%S"), "MaxDepth: " .. tostring(maxDepth), "================================", "" }
    traverse(appElem, 0, maxDepth, lines)
    writeFile(outPath, lines)
end

function M.dumpWindow(appName, windowIndex, maxDepth)
    appName = appName or "Finder"; windowIndex = windowIndex or 1; maxDepth = maxDepth or 10
    local appElem = getAppElement(appName)
    if not appElem then return end
    local ok, windows = pcall(function() return appElem:attributeValue("AXWindows") or {} end)
    if not ok or not windows or #windows == 0 then print("[ax-dump] No windows"); return end
    if windowIndex > #windows then
        print("[ax-dump] Window index out of range. Available:")
        for i, win in ipairs(windows) do print(string.format("  [%d] %s", i, win:attributeValue("AXTitle") or "(no title)")) end
        return
    end
    local winElem  = windows[windowIndex]
    local winTitle = winElem:attributeValue("AXTitle") or "(no title)"
    local ts = timestamp()
    local outPath = RAW_DIR .. "/ax-dump-" .. appName .. "-win" .. windowIndex .. "-" .. ts .. ".txt"
    local lines = { "=== AX Window Dump ===", "App: " .. appName, "Window[" .. windowIndex .. "]: " .. winTitle, "Date: " .. os.date("%Y-%m-%d %H:%M:%S"), "MaxDepth: " .. tostring(maxDepth), "================================", "" }
    traverse(winElem, 0, maxDepth, lines)
    writeFile(outPath, lines)
end

return M
```

## ~/.hammerspoon/init.lua에 추가할 코드

```lua
-- AI Memory Agent modules
package.path = os.getenv("HOME") .. "/{project}/agent/hammerspoon/?.lua;" .. package.path

local watcher   = require("watcher")
local kakaotalk = require("kakaotalk")
local axDump    = require("ax-dump")

watcher.start()
kakaotalk.startTimer(30)
```
