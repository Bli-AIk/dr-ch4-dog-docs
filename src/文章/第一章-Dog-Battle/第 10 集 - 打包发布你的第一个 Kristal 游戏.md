# 第 10 集：打包发布你的第一个 Kristal 游戏

欢迎回来！这是《从零做一个 Deltarune 同人游戏》系列的第 10 集，也是**第一章的最后一集**。

第 04 集，项目跑起来、进入第一场战斗；第 05-09 集，敌人、三回合弹幕、ACT 与饶恕——一场 Dog Battle 该有的都有了。但这些东西都还在你自己的电脑上。这一集把"能玩"变成"能交付"：**打包发布**，让别人也能玩到。

交付两种东西，对应两种玩家：

- **mod 包**——给已经装了 Kristal 的人。一个 zip，丢进 `mods/` 就能玩；
- **独立包**——给什么都不想装的人。一个可以直接双击的 exe。

第 03 集清单的最后一行"打包发布"，就是这一集。

## 发布前整理项目

打包不是"把文件夹压一下"，而是**决定什么进包**。判断标准只有一条：**运行时需要吗？**

EX04 已经把项目分好了三类：源文件（要发布）、构建产物（不进包）、个人配置（不进包）。发布前照例体检一遍，用的都是学过的命令：

- `git status` 干净（EX03）——该提交的提交了，不该在的别在；
- 构建产物 `dist/`、`.build/` 都在 `.gitignore` 里（EX04）；
- 素材出处 `THIRD_PARTY.md` 是完整的（EX04）；
- 开发库躺在 `libraries/` 里——它们怎么被拦在包外，是这一节的重点。

顺带看一眼 `mod.json` 里的 `name`、`subtitle`、`version`——主菜单按钮上显示的就是它们。发布前把它们改成正式的名字和版本号（怎么升版本，EX04 教过 SemVer）。

说具体点，我们项目的三类各是什么：

- **源文件**：`mod.json`（mod 的身份证）、`scripts/`（全部代码）、`lang/`（双语文本）、`assets/`（素材）、`libraries/`（库）——这些进包；
- **构建产物**：`dist/`、`.build/`——脚本生成的，随时能重新生成，带进包只会白占体积；
- **个人配置**：`.obsidian/`、编辑器配置——只对你有用，别人不需要。

这条分类线 EX04 已经画好，这里只是照着做减法。体检完，项目本身不用动——**剔除开发内容这件事，构建脚本替我们做好**。这也是 EX04 说"生产打包时自动排除"的含义。

## 确认运行依赖

发布之前，先想清楚：**这个 mod 跑起来需要什么？** 分三层：

1. **引擎**：Kristal v0.10.0（`mod.json` 里的 `engineVer` 字段）。玩 mod 的人得有引擎——或者你给他独立包，引擎就包在里面了；
2. **运行时库**：`kristal-i18n`（语言系统）、`gaster_blaster`（GB 炮的子弹库）——缺了它们战斗跑不起来；
3. **开发库**：`object-editor`、`terminal-cli`、`kristal-debug-tools`——开发期才有用。

第 3 层是怎么被"禁用"的？看 `kristal-debug-tools` 的 `lib.json`：

```json
{
    "id": "kristal-debug-tools",
    "version": "v0.1.0",
    "engineVer": "v0.10.0",
    "authors": ["Bli-AIk"],
    "config": {
        "enabled": true,
        "only_dev": true,
        "default_encounter": null,
        "initial_tp": null,
        "initial_mercy": null
    }
}
```

关键在 `"only_dev": true`——**只在开发模式启用**。它怎么生效，看库自己的 `lib.lua`：

```lua
local function is_enabled()
    if config("enabled") == false then
        return false
    end
    if config("only_dev") == false then
        return true
    end
    -- 默认（only_dev = true）：只有开发模式才启用
    return not Kristal.isDevMode or Kristal.isDevMode()
end
```

`Kristal.isDevMode()` 看的是 `mod.json` 的 `dev` 字段。所以：**"开发模式"是 mod 自己声明的，不是引擎猜的**。发布版要把 `dev` 改成 `false`，三个开发库的功能（debug 菜单、终端、物体编辑器）就全部熄火。

这里有个容易混的点：**only_dev 只是"运行时开关"，引擎不会物理删除任何库**。库还在包里，只是不干活；"生产包整目录删掉开发库"是构建脚本的行为——下面就看到。

引擎还管两件兼容性的事。第一，`engineVer`：主菜单的 mod 按钮会做版本检查，0.x 阶段的引擎要求**完全同版本**（引擎的 semver 实现）：

```lua
  -- This works like the "pessimisstic operator" in Rubygems.
  -- if a and b are versions, a ^ b means "b is backwards-compatible with a"
  -- 0.x 阶段：完全同版本才算兼容（v0.10.0 只认 v0.10.0）
  function mt:__pow(other)
    if self.major == 0 then
      return self == other
    end
    return self.major == other.major and
           self.minor <= other.minor
  end
```

版本对不上 mod 也能加载，但主菜单会红字标出差异。为什么 0.x 这么严格？因为 0.x 意味着引擎还没承诺稳定——`v0.10.0` 到 `v0.11.0` 之间可能改掉任何东西，引擎干脆要求完全同版本；等 1.0 之后，才允许"同大版本内小版本向上兼容"（就是 `__pow` 的另一半分支）。第二，库的依赖：`lib.json` 里写了 `dependencies` 的话，缺失的必填依赖会让整个 mod 加载失败——所以发布包里，`gaster_blaster` 和 `kristal-i18n` 必须原样带上。

## 准备发布版本

两条命令，两种产物。先看 **mod 包**：

```makefile
build-mod:
    @./.github/scripts/build_mod.sh
```

跑完得到 `dist/dr-ch4-dog-mod.zip`。脚本做了三件事——先按清单排除开发文件（rsync 的排除规则，这是脚本里的一段）：

```bash
rsync -a \
    --exclude='/.git/' \
    --exclude='/.github/' \
    --exclude='/.build/' \
    --exclude='/dist/' \
    --exclude='/.emacs/' \
    --exclude='/.helix/' \
    --exclude='/.vscode/' \
    --exclude='/.worktrees/' \
    --exclude='/tests/' \
    --exclude='/docs/' \
    --exclude='/Makefile' \
    --exclude='/justfile' \
    --exclude='/build_standalone.sh' \
    --exclude='/build_standalone.py' \
    --exclude='/release-please-config.json' \
    --exclude='/.release-please-manifest.json' \
    --exclude='/.gitmodules' \
    --exclude='/.gitignore' \
    --exclude='*.tiled-project' \
    --exclude='*.tiled-session' \
    "$ROOT/" "$STAGE_DIR/"
```

排除的每一类都有道理：`.git/`、`.github/` 是版本管理的东西；`.emacs/`、`.helix/`、`.vscode/` 是编辑器配置；`docs/` 是文档仓库；`Makefile`、`justfile`、`build_*` 是开发工具；`*.tiled-project` 是地图编辑器的工程文件（第二章才用得上）。然后删掉三个开发库，把 `mod.json` 的 `dev` 改成 `false`（顺带把 object-editor 的 `enabled` 也关了）：

```jsonc
    // 发布前：开发模式开着，debug 菜单、终端、物体编辑器都能用
    "dev": true,
    // 发布后：脚本把它改成 false，开发功能全部熄火
    "dev": false,
```

打包成 zip 还有个引擎级的便利——**zip 不用解压**。引擎扫描 `mods/` 时见到 `.zip` 会自动挂载（引擎 loadthread.lua）：

```lua
        local zip_id = checkExtension(path, "zip")
        if zip_id then
            local mounted_path = full_path
            full_path = combinePath(base_dir, "mods", zip_id)
            path = zip_id
            -- 把 zip 挂载进文件系统，玩家不用自己解压
            love.filesystem.mount(mounted_path, full_path)
        end
```

所以分享时一个 zip 就够了：玩家把它放进 `mods/`，主菜单直接出现你的 mod。

再来看 **独立包**：

```makefile
build:
    @./build_standalone.sh
```

它会下载固定版本的 Kristal（v0.10.0），改几个配置，把引擎和你的 mod 焊成一个独立游戏——玩家不需要装 Kristal。改动配置的本质，是引擎 vendcust.lua 里的三行：

```lua
--- The ID of the "target mod" (found in mod.json).
--- 锁定的目标 mod：玩家只能玩这一个
TARGET_MOD = nil

--- Disables Kristal's built-in Main menu and
--- immediately loads the target mod.
--- 跳过主菜单，启动直接进你的 mod
AUTO_MOD_START = false

--- Controls whether Kristal development-related features are enabled or not.
--- 正式版：开发功能关闭
RELEASE_MODE = true
```

（上面的 `nil`/`false`/`true` 是引擎默认值，构建脚本把它们改成 `"dr-ch4-dog"` / `true` / `true`。）再改 `conf.lua`：`identity`（存档目录——改了才不会和引擎本体共用存档）、窗口标题和图标。最后把 `love.exe` 和打包好的 `.love` 拼在一起——`DR-CH4-DOG-release.exe` 诞生。产物汇总：

| 产物 | 给谁 | 怎么用 |
| --- | --- | --- |
| `dr-ch4-dog-mod.zip` | 装了 Kristal 的人 | 丢进 `mods/` 文件夹 |
| `DR-CH4-DOG-release-win64.zip` | 任何人 | 解压后双击 exe |
| `dr-ch4-dog-release.love` | 装了 LÖVE 的人 | 拖进 love.exe |

构建脚本默认出两套独立包：`release` 和 `debug`。release 是给玩家的（开发功能全关）；debug 是留给自己的调试版（窗口标题带 " Debug"、mod 名带 `_debug` 后缀）——**发布时别把 debug 变体发出去**，它只留着开发用。

什么时候给哪种？mod zip 适合圈内分享和快速迭代；独立包适合正式发布——门槛最低，一个 exe 解决一切。

## 测试发布包

发布前最后一分钟检查：**用玩家视角跑一遍**（EX06 冒烟测试的思路）。别用你自己的开发环境，装进一个干净的 Kristal：

1. 把 `dr-ch4-dog-mod.zip` 放进干净 Kristal 的 `mods/` 文件夹；
2. 启动，主菜单出现 dr-ch4-dog（名字、副标题、版本号就在按钮上）；
3. 进战斗：三回合弹幕、ACT、饶恕胜利；
4. 故意输一局，确认失败流程正常（全灭 → 游戏结束）；
5. 切一下语言——发布包里 en 和 zh_hans 都在，切换要正常（kristal-i18n 默认跟随系统语言）；
6. 退出，确认存档和设置没被污染。

独立包额外检查：窗口标题和图标是自己的；`Shift+`` 的 debug 菜单没了（RELEASE_MODE 生效）；存档目录独立（identity 改了，不会和引擎本体共用存档）。

这一分钟最划算——电脑上多放一个干净的 Kristal 不费事，但"发出去才发现跑不了"就尴尬了。

> 📷 此处插入截图：干净 Kristal 主菜单上的 dr-ch4-dog 条目——名字、副标题、版本号都在按钮上

## 整理作品说明

发布 = 让陌生人能自己上手。他不知道你的项目，README 就是说明书。给它配一个能读懂的 README 骨架：

```markdown
# dr-ch4-dog

一个 Deltarune 同人战斗：三回合神烦狗战，一条饶恕之路。

## 运行方式

需要 Kristal v0.10.0（kristal.cc/wiki/downloading）：
把 `dr-ch4-dog-mod.zip` 放进 Kristal 的 `mods/` 文件夹，
启动后在主菜单选择 dr-ch4-dog。

## 素材与许可

素材出处与许可见 THIRD_PARTY.md。
```

README 里顺手写明版本和更新说明（"v0.1.0：第一个可玩版本"），玩家才知道自己拿到的是什么。再确认一眼 `THIRD_PARTY.md`（EX04 教过怎么登记）——发布时它是你的底气：每个素材都有出处。

## 发布与回顾

最后一步，把包发出去。EX03 教过 tag 和 CI，EX04 教过 release-please 的 release PR——现在把它们接起来：合并 release PR → 机器人打 `v0.1.0` 标签 → CI 自动构建 → 产物自动上传到 **GitHub Releases 页面**：`.love`、`win64` 包、`mod.zip`，还有一份 `SHA256SUMS`。

`SHA256SUMS` 是校验清单——下载的人跑一条命令，就能确认文件没坏、没被替换：

```bash
sha256sum -c SHA256SUMS
```

GitHub Releases 页面长这样：一个版本一个条目，标题就是版本号（v0.1.0），下面是 changelog 和附件列表——玩家在页面上挑自己平台的包下载，顺手还能跑一遍校验。

> 📷 此处插入截图：GitHub Releases 页面——v0.1.0 条目与附件列表

到这里，第 03 集清单的最后一行也打上勾了。回看第一章：规划（第 03 集）→ 战斗闭环（第 04-09 集）→ 交付（这一集）——一个从零做出来、别人能玩到的 Dog Battle。

## 结尾

第一章收官。从第 04 集那个启动画面，到现在的 exe——你做出了一个完整的、可以交付的游戏。

下一章，第二章：Overworld——用 Tiled 制作第一张地图，让狗从战斗走进地图，把故事串起来。

你也可以试试看：

- 把 mod.zip 装进一个干净的 Kristal（或者借朋友的电脑）跑一遍完整流程——模拟真实玩家，最容易找到"我以为没问题"的坑；
- 给独立包换个身份：改 `mod.json` 的 `name` 和 `subtitle`，重新 `just build`，看窗口标题和主菜单的变化。
