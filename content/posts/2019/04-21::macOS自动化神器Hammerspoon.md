+++
title = "macOS自动化神器Hammerspoon"
date = "2019-04-21 16:11:00"
url = "archives/557"
tags = ["macOS"]
categories = ["杂项"]
+++

Hammerspoon 是 macOS 上一个强大的自动化工具，是一款开源软件，但安装之后 Hammerspoon 默认什么功能也没有，所有的功能都在 Lua 脚本中，需要用户自己编写。Hammerspoon 可以让用户通过 Lua 脚本直接调用 macOS 提供的 API，能做的事情既有自定义快捷键这种简单操作，也能实现连上家里 Wi-Fi 后自动打开某视频网站、到办公室后自动静音等复杂功能。官方提供的 API 已经非常丰富，包括管理应用程序、管理系统音频设备、画图、网格化窗口、控制系统电源状态、操纵显示屏、控制鼠标、执行HTTP请求、HTTP服务器、执行shell/applescript/javascript代码等等。

## 安装 ##

安装非常简单，这当然是直接使用`Homebrew`，执行`brew install hammerspoon`即可。启动 Hammerspoon 后，程序会自动加载配置，配置文件位于 `~/.hammerspoon/init.lua`

## 使用 ##

在使用之前，你需要花几分钟简单的学习一下`lua语言`，[http://www.runoob.com/lua/lua-tutorial.html][http_www.runoob.com_lua_lua-tutorial.html]  
然后对照着hammerspoon提供的api，实现你想实现的一切功能。[http://www.hammerspoon.org/docs/][http_www.hammerspoon.org_docs]

如果你不想写，也可以看看其他大神提供的脚本，看看都实现了些什么功能，拿来改吧改吧也能用呢。  
[https://github.com/search?q=hammerspoon][https_github.com_search_q_hammerspoon]

## 实战 ##

这里我先参考文档，和其他脚本，实现了几个小功能（显示天气预报/网速监控/内存监控/窗口管理）

![Jietu20190425-213726-HD.gif][]![image.png][]![Jietu20190425-213952-HD.gif][]![1123123123.gif][]

首先是快捷键绑定，由于开发阶段需要频繁的加载配置文件，每次都去找到Hammerspoon然后点一下Relod Config还是比较烦人的事情。

```lua
local hotkey = require('core.hotkey')

hotkey.bindWithCtrlCmdAlt('R', '重新加载配置文件', function()
    hs.alert.show('加载配置文件中..')
    hs.timer.doAfter(0.1, function()hs.reload()end)
end)
```

这里的`core.hotkey`是对`hs.hotkey`的封装，代码如下：

```lua
hotkey = {
    registeredHotkey = {}
}
local strkit = require('core.strkit')

function hotkey.bind(mods, key, desc, fn)
    hs.hotkey.bind(mods, key, fn)

    --///////////注册快捷键////////////////
    local info = ''
    for _, k in pairs(mods) do
        info = info .. (info ~= '' and '+' or '') .. strkit.firstUp(k)
    end
    info = (info .. '+' .. strkit.firstUp(key))

    table.insert(hotkey.registeredHotkey, {
        key = info,
        desc = desc
    })
    hs.printf('[注册快捷键]%s -> %s', info, desc)
end

function hotkey.bindWithCtrl(key, desc, fn)
    hotkey.bind({ 'CTRL'}, key, desc, fn)
end

function hotkey.bindWithCmd(key, desc, fn)
    hotkey.bind({ 'CMD'}, key, desc, fn)
end

function hotkey.bindWithShift(key, desc, fn)
    hotkey.bind({ 'Shift'}, key, desc, fn)
end

function hotkey.bindWithAlt(key, desc, fn)
    hotkey.bind({ 'Alt'}, key, desc, fn)
end

function hotkey.bindWithCmdAlt(key, desc, fn)
    hotkey.bind({ 'CMD', 'ALT' }, key, desc, fn)
end


function hotkey.bindWithCtrlCmd(key, desc, fn)
    hotkey.bind({ 'CTRL', 'CMD' }, key, desc, fn)
end

function hotkey.bindWithCtrlCmdAlt(key, desc, fn)
    hotkey.bind({ 'CTRL', 'CMD', 'ALT' }, key, desc, fn)
end

function hotkey.bindWithCtrlAlt(key, desc, fn)
    hotkey.bind({ 'CTRL', 'ALT' }, key, desc, fn)
end

function hotkey.bindWithCtrlShift(key, desc, fn)
    hotkey.bind({ 'CTRL', 'SHIFT' }, key, desc, fn)
end

function hotkey.bindWithCtrlShiftCmd(key, desc, fn)
    hotkey.bind({ 'CTRL', 'SHIFT', 'CMD' }, key, desc, fn)
end

function hotkey.bindWithCtrlShiftAlt(key, desc, fn)
    hotkey.bind({ 'CTRL', 'SHIFT', 'ALT' }, key, desc, fn)
end

function hotkey.bindWithShiftAlt(key, desc, fn)
    hotkey.bind({ 'SHIFT', 'ALT' }, key, desc, fn)
end

function hotkey.bindWithShiftCmd(key, desc, fn)
    hotkey.bind({ 'SHIFT', 'CMD' }, key, desc, fn)
end

function hotkey.bindWithShiftCmdAlt(key, desc, fn)
    hotkey.bind({ 'SHIFT', 'CMD', 'ALT' }, key, desc, fn)
end

hotkey.bindWithCtrlCmdAlt('K', '显示所有快捷键', function()
    allHotKey = ""
    for _, v in pairs(hotkey.registeredHotkey) do
        allHotKey = allHotKey .. '▶︎ (' .. v.key .. ') ' .. v.desc .. '\n'
    end
    hs.dialog.blockAlert("已注册的快捷键", allHotKey, "我知道了")
end)

return hotkey
```

然后进行模块划分，后续可能会加很多功能进来， 不可能全部写在`~/.hammerspoon/init.lua`文件中。  
![image.png][image.png 1]

在`init.lua`文件中，按需加载即可。

```lua
require('modules.default')
require('modules.reload')
require('modules.config')
require('modules.hosts')
require('modules.lockscreen')
require('modules.weather')
require('modules.speed')
require('modules.memory')
require('modules.window')

--// 加载私有模块
if (string.find(hs.execute('whoami'), 'wuwenze') ~= nil) then
    require('private.helpdesk')
end
```

### 天气预报 ###

```lua
local apiUrl = "https://www.tianqiapi.com/api/?version=v1"

local weatherIcon = {
    loading = hs.image.imageFromPath('assets/weather/loading.ico'):setSize({ w = 20, h = 20 }),
    reload = hs.image.imageFromPath('assets/weather/reload.ico'):setSize({ w = 20, h = 20 }),
    lei = hs.image.imageFromPath('assets/weather/lei.ico'):setSize({ w = 20, h = 20 }),
    qing = hs.image.imageFromPath('assets/weather/qing.ico'):setSize({ w = 20, h = 20 }),
    wu = hs.image.imageFromPath('assets/weather/wu.ico'):setSize({ w = 20, h = 20 }),
    xue = hs.image.imageFromPath('assets/weather/xue.ico'):setSize({ w = 20, h = 20 }),
    yu = hs.image.imageFromPath('assets/weather/yu.ico'):setSize({ w = 20, h = 20 }),
    yujiaxue = hs.image.imageFromPath('assets/weather/yujiaxue.ico'):setSize({ w = 20, h = 20 }),
    yun = hs.image.imageFromPath('assets/weather/yun.ico'):setSize({ w = 20, h = 20 }),
    zhenyu = hs.image.imageFromPath('assets/weather/zhenyu.ico'):setSize({ w = 20, h = 20 }),
    yin = hs.image.imageFromPath('assets/weather/yin.ico'):setSize({ w = 20, h = 20 }),
    xiaoyu = hs.image.imageFromPath('assets/weather/xiaoyu.ico'):setSize({ w = 20, h = 20 }),
    bingbao = hs.image.imageFromPath('assets/weather/bingbao.ico'):setSize({ w = 20, h = 20 }),
    taifeng = hs.image.imageFromPath('assets/weather/taifeng.ico'):setSize({ w = 20, h = 20 }),
    shachen = hs.image.imageFromPath('assets/weather/shachen.png'):setSize({ w = 20, h = 20 }),
}

local weatherBar = hs.menubar.new()
weatherBar:setIcon(weatherIcon['loading'])
-- weatherBar:setTitle('查询天气中..')

local reloadItem = {
    title = '重新加载',
    featuredImage = weatherIcon['reload'],
    fn = function() fetchWeatherInfo(true) end
}

local function fetchWeatherInfo(isReload)
    hs.http.asyncGet(apiUrl, nil, function(status, body, _)
        if status ~= 200 then
            hs.alert.show('fetchWeatherInfo error, status = ' .. status)
            return
        end

        local weatherMenu = {}
        json = hs.json.decode(body)
        for i, v in pairs(json.data) do
            if i == 1 then
                -- weatherBar:setTitle(v.wea)
                weatherBar:setIcon(weatherIcon[v.wea_img])
            end

            weatherMenu[i] = {
                featuredImage = weatherIcon[v.wea_img],
                title = string.format('%s%s %s %s', v.day, v.wea, v.tem, v.win_speed)
            }
        end
        table.insert(weatherMenu, reloadItem)
        weatherBar:setMenu(weatherMenu)

        if (isReload) then
            hs.alert.show(string.format('%s 天气预报已更新', json.city))
        end
    end)
end

fetchWeatherInfo(false)
```

### 内存监控 ###

```lua
local memoryIcon = {
    icon = hs.image.imageFromPath('assets/memory/icon.png'):setSize({ w = 20, h = 20 }),
    clean = hs.image.imageFromPath('assets/memory/clean.png'):setSize({ w = 20, h = 20 }),
}
local fetchTimer = nil
local isCleaning = false
local memoryBar = hs.menubar.new()

memoryBar:setTitle('0.00%') --used_rate
memoryBar:setIcon(memoryIcon['icon'])
memoryBar:setTooltip('0M used (0M wired), 0M unused')
memoryBar:setClickCallback(function()
    isCleaning = true
    if (fetchTimer ~= nil) then
        fetchTimer:stop()
    end

    memoryBar:setTitle('清理中..')
    memoryBar:setIcon(memoryIcon['clean'])
    hs.execute('sudo purge')
    if (fetchTimer ~= nil) then
        fetchTimer:start()
    end
end)

local function fetchPhysMem()
    if (isCleaning) then
        memoryBar:setIcon(memoryIcon['icon'])
        isCleaning = false
    end
    
    -- PhysMem: 9032M used (2148M wired), 7351M unused.
    physMem = hs.execute('top -l 1 | head -n 10 | grep PhysMem')
    
    -- 9032M used
    used_text = string.match(physMem, '[%d]+%a used')
    wired_text = string.match(physMem, '[%d]+%a wired')
    unused_text = string.match(physMem, '[%d]+%a unused')
    
    -- M or G
    used_unit = string.match(used_text, '%u')
    wired_unit = string.match(wired_text, '%u')
    unused_unit = string.match(unused_text, '%u')
    
    -- 9032
    used = string.match(used_text, '[%d]+')
    if (used_unit == 'G') then
        used = used * 1024
    end
    wired = string.match(wired_text, '[%d]+')
    if (wired_unit == 'G') then
        wired = wired * 1024
    end
    unused = string.match(unused_text, '[%d]+')
    if (unused_unit == 'G') then
        unused = unused * 1024
    end
    
    -- used_rate = (used - wired) / (used + unused) ?
    used_rate =  used / (used + unused)
    memoryBar:setTitle((string.gsub(string.format("%6.0f", used_rate * 100), "^%s*(.-)%s*$", "%1"))..'%')
    memoryBar:setTooltip(string.format('%dM used (%dM wired), %dM unused',used, wired, unused))
end
fetchTimer = hs.timer.doEvery(5, fetchPhysMem)
fetchPhysMem()
```

### 窗口管理 ###

```lua
local hotkey = require('core.hotkey')
local function move(x, y, w, h)
    return function()
        win = hs.window.focusedWindow()
        if win then
            win_f = win:frame()
            screen_f = win:screen():frame()
            print('input: x='..x..',y='..y..',w='..w..',h='..h)
            print('screen: x='..screen_f.x..',y='..screen_f.y..',w='..screen_f.w..',h='..screen_f.h)
            print('begin_window: x='..win_f.x..',y='..win_f.y..',w='..win_f.w..',h='..win_f.h)
            print('win_f.x = screen_f.w * x + screen_f.x -> '.. screen_f.w * x + screen_f.x)
            print('win_f.y = screen_f.h * y -> '.. screen_f.h * y)
            print('win_f.w = screen_f.w * w -> '.. screen_f.w * w)
            print('win_f.h = screen_f.h * h -> '.. screen_f.h * h)

            win_f.x = screen_f.w * x + screen_f.x
            win_f.y = screen_f.h * y
            win_f.w = screen_f.w * w
            win_f.h = screen_f.h * h

            print('end_window: x='..win_f.x..',y='..win_f.y..',w='..win_f.w..',h='..win_f.h)
            win:setFrame(win_f, 0)
        end
    end
end

hotkey.bindWithCtrlShift('Up', '[窗口管理]向上移动窗口', move(0, 0, 1, 0.5))
hotkey.bindWithCtrlShift('Right', '[窗口管理]向右移动窗口', move(0.5, 0, 0.5, 1))
hotkey.bindWithCtrlShift('Down', '[窗口管理]向下移动窗口', move(0, 0.5, 1, 0.5))
hotkey.bindWithCtrlShift('Left', '[窗口管理]向左移动窗口', move(0, 0, 0.5, 1))
hotkey.bindWithCtrlShift('M', '[窗口管理]最大化窗口', move(0, 0, 1, 1))
hotkey.bindWithCtrlShift('C', '[窗口管理]居中窗口', move(0.05, 0.08, 0.9, 0.9))
hotkey.bindWithCmdAlt('Left', '[窗口管理]向左上角移动窗口', move(0, 0, 0.5, 0.5))
hotkey.bindWithShiftCmdAlt('Left', '[窗口管理]向左下角移动窗口', move(0, 0.5, 0.5, 0.5))
hotkey.bindWithCmdAlt('Right', '[窗口管理]向右上角移动窗口', move(0.5, 0, 0.5, 0.5))
hotkey.bindWithShiftCmdAlt('Right', '[窗口管理]向右下角移动窗口', move(0.5, 0.5, 0.5, 0.5))
```

### 网速监控 ###

```lua
local speedBar = hs.menubar.new()
speedBar:setTitle('0.00 KB/s')
speedBar:setIcon(hs.image.imageFromPath('assets/speed/down.ico'):setSize({ w = 20, h = 20 }))

local interface = hs.network.primaryInterfaces();
if interface then
    local netstat_down = 'netstat -ibn | grep -e ' .. interface .. ' -m 1 | awk \'{print $7}\''
    local netstat_up = 'netstat -ibn | grep -e ' .. interface .. ' -m 1 | awk \'{print $10}\''
    local prev_speed_down = hs.execute(netstat_down)
    local prev_speed_up = hs.execute(netstat_up)

    hs.timer.doEvery(1, function()
        speed_down = hs.execute(netstat_down)
        speed_up = hs.execute(netstat_up)
        speed_down_show = format_show(speed_down - prev_speed_down)
        speed_up_show = format_show(speed_up - prev_speed_up)
        prev_speed_down = speed_down
        prev_speed_up = speed_up
        speedBar:setTitle(speed_down_show)
        speedBar:setTooltip('UP:'..speed_up_show..', DOWN:'..speed_down_show)
    end)

    function format_show(diff)
        if diff/1024 > 1024 then
            return trim(string.format("%6.2f MB/s", diff/1024/1024))
        end
        return trim(string.format("%6.2f KB/s", diff/1024))
    end

    function trim (s)
        return (string.gsub(s, "^%s*(.-)%s*$", "%1"))
    end
end
```

有了如此强大开放的工具，我还装那些乱七八糟的辅助软件干什么呢？自己实现一个，才是最极客的方式！

脚本地址：[https://gitee.com/wuwenze/hammerspoon-config][https_gitee.com_wuwenze_hammerspoon-config]，后续我会持续更新新的功能进来的，哈哈。


[http_www.runoob.com_lua_lua-tutorial.html]: http://www.runoob.com/lua/lua-tutorial.html
[http_www.hammerspoon.org_docs]: http://www.hammerspoon.org/docs/
[https_github.com_search_q_hammerspoon]: https://github.com/search?q=hammerspoon
[Jietu20190425-213726-HD.gif]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/1556199473567-8ae7cc91-c96e-4788-933c-e1d9548267c4.gif
[image.png]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/1556199714274-50a04b65-c6f2-4854-a81f-ee9b79689fd6.png
[Jietu20190425-213952-HD.gif]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/33333333.gif
[1123123123.gif]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/444444444444.gif
[image.png 1]: https://wuwenze-1253645550.cos.ap-chengdu.myqcloud.com/2019-08-19-073551.png
[https_gitee.com_wuwenze_hammerspoon-config]: https://gitee.com/wuwenze/hammerspoon-config