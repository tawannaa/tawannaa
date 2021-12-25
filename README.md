--[[
Change logs:

12/5/21
    * Fixed issues with noteTime calculation, causing some songs like Triple Trouble to break. Report bugs as always

11/9/21
    + Added support for new modes (9Key for example)

9/26/21 
    + Added 'Unload'
    * Fixed issues with accuracy.

9/25/21 (patch 1)
    * Added a few sanity checks
    * Fixed some errors
    * Should finally fix invisible notes (if it doesnt, i hate this game)

9/25/21
    * Code refactoring.
    * Fixed unsupported exploit check
    * Implemented safer URL loading routine.
    * Tweaked autoplayer (implemented hitbox offset, uses game code to calculate score and hit type now)

9/19/21
   * Miss actually ignores the note.

8/20/21
   ! This update was provided by Sezei (https://github.com/greasemonkey123/ff-bot-new)
       * I renamed some stuff and changed their default 'Autoplayer bind'

   + Added 'Miss chance'
   + Added 'Release delay' (note: higher values means a higher chance to miss)
   + Added 'Autoplayer bind'
   * Added new credits
   * Made folder names more clear

8/2/21
    ! KRNL has since been fixed, enjoy!

    + Added 'Manual' mode which allows you to force the notes to hit a specific type by holding down a keybind.
    * Switched fastWait and fastSpawn to Roblox's task libraries
    * Attempted to fix 'invalid key to next' errors

5/12/21
    * Attempted to fix the autoplayer missing as much.

5/16/21
    * Attempt to fix invisible notes.
    * Added hit chances & an autoplayer toggle
    ! Hit chances are a bit rough but should work.

Information:
    Officially supported: Synapse X, Script-Ware, KRNL, Fluxus
    Needed functions: setthreadcontext, getconnections, getgc, getloaodedmodules 

    You can find contact information on the GitHub repository (https://github.com/wally-rblx/funky-friday-autoplay)
--]]

local client = game:GetService('Players').LocalPlayer;
local set_identity = (type(syn) == 'table' and syn.set_thread_identity) or setidentity or setthreadcontext

local function fail(r) return client:Kick(r) end

-- gracefully handle errors when loading external scripts
local function urlLoad(url)
    local success, result = pcall(game.HttpGet, game, url)
    if (not success) then
        return fail(string.format('Failed to GET url %q for reason: %q', url, tostring(result)))
    end

    local fn, err = loadstring(result)
    if (type(fn) ~= 'function') then
        return fail(string.format('Failed to loadstring url %q for reason: %q', url, tostring(err)))
    end

    local results = { pcall(fn) }
    if (not results[1]) then
        return fail(string.format('Failed to initialize url %q for reason: %q', url, tostring(results[2])))
    end

    return unpack(results, 2)
end

-- attempt to block imcompatible exploits
-- rewrote because old checks literally did not work
if type(set_identity) ~= 'function' then return fail('Unsupported exploit (missing "set_thread_identity")') end
if type(getconnections) ~= 'function' then return fail('Unsupported exploit (missing "getconnections")') end
if type(getloadedmodules) ~= 'function' then return fail('Unsupported exploit (misssing "getloadedmodules")') end
if type(getgc) ~= 'function' then return fail('Unsupported exploit (misssing "getgc")') end

local library = urlLoad("https://raw.githubusercontent.com/wally-rblx/uwuware-ui/main/main.lua")

local framework, scrollHandler
local counter = 0

while true do
    for _, obj in next, getgc(true) do
        if type(obj) == 'table' and rawget(obj, 'GameUI') then
            framework = obj;
            break
        end 
    end

    for _, module in next, getloadedmodules() do
        if module.Name == 'ScrollHandler' then
            scrollHandler = module;
            break;
        end
    end

    if (type(framework) == 'table') and (typeof(scrollHandler) == 'Instance') then
        break
    end

    counter = counter + 1
    if counter > 6 then
        fail(string.format('Failed to load game dependencies. Details: %s, %s', type(framework), typeof(scrollHandler)))
    end
    wait(1)
end

local runService = game:GetService('RunService')
local userInputService = game:GetService('UserInputService')
local virtualInputManager = game:GetService('VirtualInputManager')

local random = Random.new()

local task = task or getrenv().task;
local fastWait, fastSpawn = task.wait, task.spawn;

-- firesignal implementation
-- hitchance rolling
local fireSignal, rollChance do
    -- updated for script-ware or whatever
    -- attempted to update for krnl

    function fireSignal(target, signal, ...)
        -- getconnections with InputBegan / InputEnded does not work without setting Synapse to the game's context level
        set_identity(2)
        local didFire = false
        for _, signal in next, getconnections(signal) do
            if type(signal.Function) == 'function' and islclosure(signal.Function) then
                local scr = rawget(getfenv(signal.Function), 'script')
                if scr == target then
                    didFire = true
                    local s, e = pcall(signal.Function, ...)
                --    if not s then fail("failed to call input: " .. tostring(e)) end
                end
            end
        end
        -- if not didFire then fail"couldnt fire input signal" end
        set_identity(7)
    end

    -- uses a weighted random system
    -- its a bit scuffed rn but it works good enough

    function rollChance()
        if (library.flags.autoPlayerMode == 'Manual') then
            if (library.flags.sickHeld) then return 'Sick' end
            if (library.flags.goodHeld) then return 'Good' end
            if (library.flags.okayHeld) then return 'Ok' end
            if (library.flags.missHeld) then return 'Bad' end

            return 'Bad' -- incase if it cant find one
        end

        local chances = {
            { type = 'Sick', value = library.flags.sickChance },
            { type = 'Good', value = library.flags.goodChance },
            { type = 'Ok', value = library.flags.okChance },
            { type = 'Bad', value = library.flags.badChance },
            { type = 'Miss' , value = library.flags.missChance },
        }

        table.sort(chances, function(a, b)
            return a.value > b.value
        end)

        local sum = 0;
        for i = 1, #chances do
            sum += chances[i].value
        end

        if sum == 0 then
            -- forgot to change this before?
            -- fixed 6/5/21

            return chances[random:NextInteger(1, #chances)].type
        end

        local initialWeight = random:NextInteger(0, sum)
        local weight = 0;

        for i = 1, #chances do
            weight = weight + chances[i].value

            if weight > initialWeight then
                return chances[i].type
            end
        end

        return 'Sick' -- just incase it fails?
    end
end

-- autoplayer
do
    local chanceValues = { 
        Sick = 96,
        Good = 92,
        Ok = 87,
        Bad = 75,
    }

    local keyCodeMap = {}
    for _, enum in next, Enum.KeyCode:GetEnumItems() do
        keyCodeMap[enum.Value] = enum
    end

    if shared._unload then
        pcall(shared._unload)
    end

    function shared._unload()
        if shared._id then
            pcall(runService.UnbindFromRenderStep, runService, shared._id)
        end

        if library.open then
            library:Close()
        end

        library.base:ClearAllChildren()
        library.base:Destroy()
    end

    shared._id = game:GetService('HttpService'):GenerateGUID(false)
    runService:BindToRenderStep(shared._id, 1, function()
        if (not library.flags.autoPlayer) then return end
        if typeof(framework.SongPlayer.CurrentlyPlaying) ~= 'Instance' then return end
        if framework.SongPlayer.CurrentlyPlaying.ClassName ~= 'Sound' then return end

        local arrows = {}
        for _, obj in next, framework.UI.ActiveSections do
            arrows[#arrows + 1] = obj;
        end

        local count = framework.SongPlayer:GetKeyCount()
        local mode = count .. 'Key'

        local arrowData = framework.ArrowData[mode].Arrows

        for idx = 1, #arrows do
            local arrow = arrows[idx]
            if type(arrow) ~= 'table' then
                continue
            end

            if type(arrow.NoteDataConfigs) == 'table' and arrow.NoteDataConfigs.Type == 'Death' then 
                continue
            end

            if (arrow.Side == framework.UI.CurrentSide) and (not arrow.Marked) and framework.SongPlayer.CurrentlyPlaying.TimePosition > 0 then
                local indice = (arrow.Data.Position % count)
                local position = indice .. ''

                if (position) then
                    local hitboxOffset = 0 do
                        local settings = framework.Settings;
                        local offset = type(settings) == 'table' and settings.HitboxOffset;
                        local value = type(offset) == 'table' and offset.Value;

                        if type(value) == 'number' then
                            hitboxOffset = value;
                        end

                        hitboxOffset = hitboxOffset / 1000
                    end

                    local songTime = framework.SongPlayer.CurrentTime do
                        local configs = framework.SongPlayer.CurrentSongConfigs
                        local playbackSpeed = type(configs) == 'table' and configs.PlaybackSpeed

                        if type(playbackSpeed) ~= 'number' then
                            playbackSpeed = 1
                        end

                        songTime = songTime /  playbackSpeed
                    end

                    local noteTime = math.clamp((1 - math.abs(arrow.Data.Time - (songTime + hitboxOffset))) * 100, 0, 100)

                    local result = rollChance()
                    arrow._hitChance = arrow._hitChance or result;

                    local hitChance = (library.flags.autoPlayerMode == 'Manual' and result or arrow._hitChance)
                    if hitChance ~= "Miss" and noteTime >= chanceValues[arrow._hitChance] then
                        fastSpawn(function()
                            arrow.Marked = true;
                            local keyCode = keyCodeMap[arrowData[position].Keybinds.Keyboard[1]]

                            if library.flags.secondaryPressMode then
                                virtualInputManager:SendKeyEvent(true, keyCode, false, nil)
                            else
                                fireSignal(scrollHandler, userInputService.InputBegan, { KeyCode = keyCode, UserInputType = Enum.UserInputType.Keyboard }, false)
                            end

                            if arrow.Data.Length > 0 then
                                fastWait(arrow.Data.Length + (library.flags.autoDelay / 1000))
                            else
                                fastWait(library.flags.autoDelay / 1000)
                            end

                            if library.flags.secondaryPressMode then
                                virtualInputManager:SendKeyEvent(false, keyCode, false, nil)
                            else
                                fireSignal(scrollHandler, userInputService.InputEnded, { KeyCode = keyCode, UserInputType = Enum.UserInputType.Keyboard }, false)
                            end

                            arrow.Marked = nil;
                        end)
                    end
                end
            end
        end
    end)
end

-- menu
do
    local window = library:CreateWindow('auto x hub Funky') do
        local folder = window:AddFolder('Autoplayer') do
            local toggle = folder:AddToggle({ text = 'Autoplayer', flag = 'autoPlayer' })

            folder:AddToggle({ text = 'Secondary press mode', flag = 'secondaryPressMode' }) -- alternate mode if something breaks on krml or whatever
            folder:AddLabel({ text = "Enable if autoplayer breaks" })
            
            -- Fixed to use toggle:SetState
            folder:AddBind({ text = 'Autoplayer toggle', flag = 'autoPlayerToggle', key = Enum.KeyCode.End, callback = function()
                toggle:SetState(not toggle.state)
            end })

            folder:AddDivider()
            folder:AddList({ text = 'Autoplayer mode', flag = 'autoPlayerMode', values = { 'Chances', 'Manual'  } })
            folder:AddDivider()
            folder:AddSlider({ text = 'Sick %', flag = 'sickChance', min = 0, max = 100, value = 100 })
            folder:AddSlider({ text = 'Good %', flag = 'goodChance', min = 0, max = 100, value = 0 })
            folder:AddSlider({ text = 'Ok %', flag = 'okChance', min = 0, max = 100, value = 0 })
            folder:AddSlider({ text = 'Bad %', flag = 'badChance', min = 0, max = 100, value = 0 })
            folder:AddSlider({ text = 'Miss %', flag = 'missChance', min = 0, max = 100, value = 0 })
            folder:AddSlider({ text = 'Release delay (ms)', flag = 'autoDelay', min = 0, max = 350, value = 50 })
        end

        local folder = window:AddFolder('Manual keybinds') do
            folder:AddBind({ text = 'Sick', flag = 'sickBind', key = Enum.KeyCode.One, hold = true, callback = function(val) library.flags.sickHeld = (not val) end, })
            folder:AddBind({ text = 'Good', flag = 'goodBind', key = Enum.KeyCode.Two, hold = true, callback = function(val) library.flags.goodHeld = (not val) end, })
            folder:AddBind({ text = 'Ok', flag = 'okBind', key = Enum.KeyCode.Three, hold = true, callback = function(val) library.flags.okayHeld = (not val) end, })
            folder:AddBind({ text = 'Bad', flag = 'badBind', key = Enum.KeyCode.Four, hold = true, callback = function(val) library.flags.missHeld = (not val) end, })
        end

        local folder = window:AddFolder('Credits') do
            folder:AddLabel({ text =      'TAWAN' })
        end

        window:AddLabel({ text = 'Version 0.2e' })
        window:AddLabel({ text = 'Updated 25/12/21' })
      
        window:AddDivider()
        window:AddButton({ text = 'Unload script', callback = function()
            shared._unload()
        end })
        window:AddButton({ text = 'Copy discord', callback = function()
              setclipboard("https://discord.gg/6RQJJAq65T")
        end })
        window:AddDivider()
        window:AddBind({ text = 'Menu toggle', key = Enum.KeyCode.Delete, callback = function() library:Close() end })
    end

    library:Init()
end
