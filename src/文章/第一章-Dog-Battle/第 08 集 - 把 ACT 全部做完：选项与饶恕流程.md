# 第 08 集：把 ACT 全部做完：选项与饶恕流程

欢迎回来！这是《从零做一个 Deltarune 同人游戏》系列的第 8 集，也是**第一章的第五集**。

第 07 集，第二回合的演出和炮击跑通了——狗转圈、吸收粒子、一发 GB 从天而降。但战斗还"结束不了"：狗打不死，摸也摸不着，ACT 菜单里只有"查看"能用。这一集，把交互全做完：苏西的反应、四个 ACT 选项，还有唯一的饶恕流程——战斗第一次可以真正结束。

第 03 集时间轴里"摸狗失败 → 第二回合 → 苏西反应 + 讲笑话 → 摸狗 + 饶恕 → 片尾"这一段，就是这集的内容。

> 📷 此处插入截图：ACT 菜单全开——查看、摸狗、讲笑话、队伍版摸狗

## 队友受伤了，苏西得有反应

第二回合打完，苏西会开口——根据队友受伤情况来说话。

原动画只考虑到了"无伤"的情况——苏西自然会很兴奋很高兴。但队友受伤时如果还说一句"这太酷了"——就太怪了吧啊喂？！

**队友受伤了，苏西得有反应**。这炮打完，谁倒下了、谁被轰到了，苏西都要说句话——而说什么，要看谁挨了炮。

但还有个前提是她自己还活着：要是这一炮把苏西也轰倒了，那她自然说不了话，对吧（笑）

实现分两步：在开始时记录一下全员各自的血量（这步在第 07 集的 gb.lua onStart 里，注释标着"第 08 集讲"，就是这里）；回合结束时，在 `onTurnEnd` 记下谁受伤了（顺便确认苏西还在不在）。

先看文案——三档原文在 `lang/zh_hans.json`（`battle_dog_gb_*` 三个键）：

```json
"battle_dog_gb_undamaged": "* 我去，[wait:5]大炮？！[wait:5]\n这太酷了！[wait:5]",
"battle_dog_gb_damaged": "* 靠！[wait:5][var:name]，[wait:5]\n你是在用脸接大炮吗？！[wait:5]",
"battle_dog_gb_kris_downed": "* 搞什么？！[wait:5]\n这玩意儿伤害这么高？！[wait:5]",
```

三个键正好对应三档：`undamaged` 是全员无伤（surprise_smile 表情）、`damaged` 是有人受伤（intense_angry 表情）、`kris_downed` 是有人倒下（teeth_b 表情，不管倒下的是谁都是这句，文案写死了——想更讲究可以自己改）。

文本里还藏着个之前没见过的富文本标记：`[var:name]`。它是**变量占位**——显示时会被替换成代码传进来的值。

下面代码中，先看**定义的一方**——`onTurnEnd`，回合结束时被引擎调用，`gb_narration_pending` 就是在这里定义的：

```lua
-- 回合结束时被引擎调用：记下谁受伤了，供下回合开头旁白用
function Dog:onTurnEnd()
    -- 不是 gb 回合：不记，清掉
    if self.selected_wave ~= "gb" then
        self.gb_narration_pending = false
        self.gb_party_health = nil
        self.gb_party_was_down = nil
        return
    end

    local susie = Game.battle:getPartyBattler("susie")
    -- 有没有人倒下？
    self.gb_party_was_down = false
    for _, battler in ipairs(Game.battle.party) do
        if battler.is_down then
            self.gb_party_was_down = true
            break
        end
    end

    -- 苏西还活着才有旁白（旁白是她说的，她倒了谁开口）
    self.gb_narration_pending = susie and not susie.is_down or false
    if not self.gb_narration_pending then
        self.gb_party_health = nil
        self.gb_party_was_down = nil
    end
end
```

再看**使用的一方**——`getEncounterText` 是引擎"回合开头旁白"的入口，每个回合开始要显示的文本都走这里（第 05 集 dummy 的对话也一样）：

```lua
-- 回合开头的旁白入口：返回（文案, 头像, 说话人）三个值
function Dog:getEncounterText()
    if self.gb_narration_pending then   -- 上一回合是 gb 且苏西活着，才走这套旁白
        self.gb_narration_pending = nil -- 消费掉：只报一次

        local party_health = self.gb_party_health
        local someone_was_down = self.gb_party_was_down
        self.gb_party_health = nil
        self.gb_party_was_down = nil

        -- 档位一：有人倒下——不管是谁，都是"搞什么？！这玩意儿伤害这么高？！"
        if someone_was_down then
            return Game:loc("battle_dog_gb_kris_downed"), "teeth_b", "susie"
        end

        -- 对比回合开始记的血量，找出谁被打到了
        local damaged_battlers = {}
        for _, battler in ipairs(Game.battle.party) do
            local starting_health = party_health and party_health[battler.chara.id]
            if starting_health and battler.chara:getHealth() < starting_health then
                table.insert(damaged_battlers, battler)
            end
        end

        -- 档位二：只有苏西受伤——她可不会承认，还是"搞什么？"那套
        if #damaged_battlers == 1 and damaged_battlers[1].chara.id == "susie" then
            return Game:loc("battle_dog_gb_kris_downed"), "teeth_b", "susie"
        elseif #damaged_battlers > 0 then
            -- 档位三：有人受伤——报伤得最重的那个（排除苏西，苏西只负责吐槽）
            local lowest_health_battler = nil
            for _, battler in ipairs(Game.battle.party) do
                if battler.chara.id ~= "susie"
                    and (not lowest_health_battler
                        or battler.chara:getHealth() < lowest_health_battler.chara:getHealth())
                then
                    lowest_health_battler = battler
                end
            end

            -- [name:xxx] 是名字占位，跟着语言走
            local name = lowest_health_battler
                and Game:locText("[name:" .. lowest_health_battler.chara.id .. "]")
                or Game:locText("[name:kris]")
            return Game:loc("battle_dog_gb_damaged", {name = name}), "intense_angry", "susie"
        else
            -- 档位四：全员无伤——"我去，大炮？！这太酷了！"
            return Game:loc("battle_dog_gb_undamaged"), "surprise_smile", "susie"
        end
    end

    return super.getEncounterText(self)  -- 非 gb 回合：走引擎默认旁白
end
```

代码这边就是往占位里塞值的：`Game:loc("battle_dog_gb_damaged", {name = name})` 的 `{name = name}` 把名字填进 `[var:name]`——`name` 是谁，气泡里就显示谁（名字取血量最低的那位，经 `[name:xxx]` 本地化，跟着语言走）。顺带这也是第 05 集说的 `Game:loc` 兜底用法：文本是拼好的字符串，不是台词表里现成的 `{key}`，所以在代码里解析。

这些表情名（`teeth_b`、`intense_angry`、`surprise_smile`）其实是苏西的**气泡头像**——`assets/sprites/face/susie/` 下的一张张贴图（引擎里叫 portrait）。想知道它们长啥样、还有什么别的表情：翻那个目录能查到，但更合适的路径：是游戏里按  ``Shift+` `` 打开 **debug 菜单**直接进**头像查看器**来找。

**debug菜单很值得学习，它能大幅度提升你的开发效率。**——按 `` ` `` 打开控制台直接调数值、按 **Ctrl+O** 用物体选择器拖坐标，都是调试时的常用手段。整套 debug 工具，会有一期番外专门讲。

> 📷 此处插入截图：第二回合无伤通关——"我去，大炮？！这太酷了！"

## 拆 ACT

第 05 集教过 registerAct 的基础；这集把选项补全。dog.lua 的 init 里，注册了三个 ACT（"查看"是引擎默认的，不用注册）：

```lua
-- 单人版摸狗：谁都能用
self:registerAct(self.act_pet, self.act_pet_description)
-- 队伍版摸狗：需要苏西和利艾尔斯同时在场
self:registerAct(self.act_pet_party, self.act_pet_description, {"susie", "ralsei"})
-- 讲笑话：需要苏西和利艾尔斯，还要 100 TP
self:registerAct(self.act_tell_joke, self.act_tell_joke_description, {"susie", "ralsei"}, 100)
```

第三参数是"使用这个 ACT 需要的队员"，第四个是 TP 要求。选项怎么分发？——选中 ACT 后引擎调用 `onAct`，按名字分流，全文如下：

```lua
-- ACT 选中后的分发入口：按名字分流
function Dog:onAct(battler, name)
    if name == self.act_check then
        return super.onAct(self, battler, "Check")  -- 查看：交给引擎默认

    elseif name == self.act_pet then
        local action = Game.battle:getCurrentAction()
        if action and action.party and #action.party > 0 then
            -- 队伍版摸狗：讲完笑话后摸到了！触发饶恕演出
            if self.joke_completed and not self.special_pet_completed then
                self.special_pet_completed = true
                local cutscene = Game.battle:startActCutscene("dog", "pet_party_special")
                cutscene:after(function()
                    self:spare()  -- 演出结束直接饶恕
                end)
                return
            end

            -- 没讲笑话：队伍版摸狗也会被躲开（金色 MISS）
            Game.battle:startActCutscene(function(cutscene)
                cutscene:text("{act_dog_pet_party_text}", {
                    functions = {
                        dog_pet_miss = function()
                            self:statusMessage("msg", "miss_gold")
                        end
                    }
                })
            end)
            return
        end
        return Game:loc("act_dog_pet_text")  -- 单人摸狗：一句话，被躲开

    elseif name == self.act_tell_joke then
        -- 讲笑话：放演出，结束后解锁"摸到"的条件
        local cutscene = Game.battle:startActCutscene("dog", "tell_joke")
        cutscene:after(function()
            self.joke_completed = true
            self.dialogue_override = "[instant][sound:voice/sans]"
                .. Game:loc("battle_dog_dialogue")
        end)
        return

    elseif name == "Standard" then --X-Action
        return Game:loc("act_dog_standard", {
            name = battler.chara:getName()
        })
    end

    -- If the act is none of the above, run the base onAct function
    -- (this handles the Check act)
    -- 翻译：都不是上面这些，就交给引擎默认处理（"查看"也走这里）
    return super.onAct(self, battler, name)
end
```

四个分支，三个"摸狗"、一个"讲笑话"。两个关键状态变量：`joke_completed`（讲笑话完成没）和 `special_pet_completed`（摸到过没，防重复触发）。

## 摸狗与 miss

单人摸狗最简单——直接返回一句话（`act_dog_pet_text` 原文："* 你伸手想碰[name:dog]。[wait:5]\n* 它躲开了。"）。第 03 集时间轴里的"摸狗失败"，就是它。

队伍版摸狗（没讲笑话时）走演出，气泡里藏了个新富文本标记 `[func:dog_pet_miss]`——"在这里调用函数"：

```lua
cutscene:text("{act_dog_pet_party_text}", {
    functions = {
        dog_pet_miss = function()
            self:statusMessage("msg", "miss_gold")  -- 金色 MISS
        end
    }
})
```

`act_dog_pet_party_text` 原文："* 大家一起去摸狗狗...[wait:5][func:dog_pet_miss][wait:1s]\n* 但是被它轻易地躲开了！"——`[func:xxx]` 标记会在文本播到那里时，调用 `functions` 表里对应的函数。`statusMessage("msg", "miss_gold")` 是弹一个状态气泡（金色 MISS 那一下）。

## 讲笑话

讲笑话走 `tell_joke` 演出（cutscenes/dog.lua）：苏西紧张地讲、利艾尔斯脸红地捧场、旁白 + 鼓点，最后"或许有用？"：

```lua
tell_joke = function(cutscene)
    cutscene:text("{battle_dog_tell_joke_1}")

    cutscene:text(
        "{battle_dog_tell_joke_susie}",
        "nervous",
        "susie"
    )

    cutscene:text(
        "{battle_dog_tell_joke_ralsei}",
        "blush_pleased_open",
        "ralsei",
        {
            reactions = {
                susie_reaction = {
                    "{battle_dog_tell_joke_susie_reaction}",
                    "right",
                    "bottom",
                    "nervous",
                    "susie"
                }
            }
        }
    )

    -- These lines are narration, so they intentionally have no portrait.
    -- 翻译：这两行是旁白，所以故意不带头像。
    cutscene:setSpeaker(nil)
    cutscene:text(
        "{battle_dog_tell_joke_dog}"
            .. "[wait:1s][sound:mus_rimshot]\n"
            .. "{battle_dog_tell_joke_inside}"
    )
    cutscene:text("{battle_dog_tell_joke_touch}")
end,
```

`cutscene:text(文案, 表情, 说话人)` 的第二个参数是头像表情（和 getEncounterText 返回的头像同一套）；`reactions` 让利艾尔斯说话时苏西在旁边接话。演出播完，onAct 里的 `cutscene:after` 才把 `joke_completed` 打上——狗从这时候起会"汪"（dialogue_override）。

## 队伍版摸狗与饶恕（唯一的饶恕方式）

`joke_completed` 一到位，队伍版摸狗就走完全不同的分支——`pet_party_special` 演出：狗震惊、加 100 饶恕值、白屏、睡着、漂浮道具环绕，演出结束 `spare()` 直接饶恕，战斗结束：

```lua
pet_party_special = function(cutscene)
    local dog = cutscene:getTarget()
    local shock_started = false

    cutscene:setSpeaker(nil)
    cutscene:text("{battle_dog_pet_special}", {
        skip = false,
        advance = false,
        wait = false,
        functions = {
            dog_pet_special_shock = function()
                dog:addMercy(100)             -- 饶恕值拉满
                cutscene:setAnimation(dog, "shock")  -- 狗震惊
                shock_started = true
            end
        }
    })

    -- The localized text waits after the first line, then triggers the shock.
    -- 翻译：文案播完第一行后触发震惊。
    cutscene:wait(function()
        return shock_started
    end)
    cutscene:wait(1)

    -- Keep both battler sprites shaking while the screen fades to white.
    -- 翻译：白屏淡出时两个精灵一起抖。
    dog.sprite:shake(4, 0, 0)
    dog.overlay_sprite:shake(4, 0, 0)
    local fade_out_done = cutscene:fadeOut(0.5, {
        color = COLORS.white,
        music = false
    })

    cutscene:wait(fade_out_done)

    -- Once pure white, automatically continue without waiting for input.
    -- 翻译：全白后自动继续，不等玩家按键。
    Game.battle.battle_ui:clearEncounterText()
    dog.sprite:stopShake()
    dog.overlay_sprite:stopShake()
    cutscene:setAnimation(dog, "sleep")  -- 睡着
    addFloatingProps(dog)                -- 漂浮道具环绕

    local fade_in_done = cutscene:fadeIn(0.75, {
        color = COLORS.white,
        music = false
    })
    cutscene:wait(fade_in_done)
    cutscene:text("{battle_dog_pet_special_3}")
end
```

演出本身的节奏：气泡 → 震惊 + 饶恕值拉满 → 白屏 → 睡着 + 道具飘起来 → 淡入。`cutscene:wait` 是"等一个条件/时间"；`fadeOut`/`fadeIn` 是白屏切换（`music = false` 表示不切音乐）。

## 为什么这是唯一的饶恕方式

狗是打不死的——`getAttackDamage` 直接返回 0，所有攻击它都会闪避：

```lua
-- The dog dodges every attack instead of taking damage.
-- 翻译：狗会闪避所有攻击，不掉血。
function Dog:getAttackDamage(damage, battler, points)
    return 0
end
```

没有 `onMercy` 覆写，正常摸狗摸不着，攻击全部被闪——想结束战斗，只有"讲笑话 → 队伍版摸狗"这一条路。原动画也是这个流程。

顺带把第 05 集那个参数的意义补完：队伍版摸狗和讲笑话都写了 `{"susie", "ralsei"}`——只有苏西一个人时，引擎的 `canSelectMenuItem` 逐个检查名单凑不齐，选项**变灰不可选**（看得到、点不动）；单人的摸狗（没写 party 参数）任何时候都能选。

第 03 集时间轴里"队伍版摸狗 → 饶恕 → 战斗结束"，在这一集闭环了。

## 检查本集成果

对照第 03 集清单，第 5 行的"苏西反应"部分——打勾。链路现在是：

```text
... → 玩家选 ACT → dog.lua 的 onAct 分发 → 单人摸狗被躲 / 讲笑话 / 队伍版摸狗 → 苏西的三档反应（getEncounterText） → pet_party_special → spare() → 战斗结束
```

调试：`just run --encounter dog`——直接进狗的战斗，把四个选项挨个试一遍，重点看：讲笑话后队伍版摸狗能不能正常结束战斗、只有苏西时两个队伍 ACT 是不是灰的。

> 📷 此处插入截图：队伍版摸狗成功——狗被摸到睡着，漂浮道具环绕

## 结尾

ACT 全做完，战斗能真正结束了。下一集，第三回合：车冲撞——随机弹幕、物理效果、敌人演出，还有镜头震动和爆炸。

作业：给队伍版摸狗加一个"利艾尔斯单人版"？——`registerAct` 的第三参数改成 `{"ralsei"}` 再注册一个试试，看看菜单里头像的变化，顺便验证"队伍要求"是全员在场还是任一在场。
