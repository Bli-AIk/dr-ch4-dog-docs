# 第 10 集：打包发布你的第一个 Kristal 游戏

> [[课程总览|返回课程总览]] · [[01-第一章-Dog-Battle/第 09 集 - 第三回合：随机弹幕、物理效果与敌人演出|上一集]]

## 开场

第一章收官集：把"能玩"变成"能交付"。两种交付形态——mod 包（zip 丢进 Kristal mods/ 即玩，引擎自动挂载）与独立包（exe，玩家不用装 Kristal）。对照第 03 集清单最后一行"打包发布"。

## 发布前整理项目

打包 = 决定什么进包，标准只有一条"运行时需要吗"。体检清单（git status 干净、dist/.build 在 .gitignore、THIRD_PARTY.md 完整、开发库会被构建脚本剔除）——对照 EX04 分类不重复展开；项目本身不用动，剔除由构建脚本做。

## 确认运行依赖

依赖三层：引擎（engineVer v0.10.0，0.x 要求完全同版本——semver __pow 代码块）；运行时库（kristal-i18n、gaster_blaster）；开发库（only_dev 机制：lib.json + lib.lua is_enabled 逐字；Kristal.isDevMode 看 mod.json 的 dev 字段）。澄清：only_dev 是运行时开关，引擎不物理删库；删库是构建脚本行为。lib.json dependencies 缺失 → 整个 mod 加载失败。

## 准备发布版本

just build-mod → dist/dr-ch4-dog-mod.zip：rsync 排除清单（.git/.github/编辑器配置/docs/Makefile/justfile/build 脚本/tiled 工程等）→ 删三个开发库 → patch dev=false（+kristal-object-selector-plus 禁用）；zip 自动挂载不用解压（loadthread.lua 片段）。just build → 独立包：固定引擎 v0.10.0 + vendcust.lua 三行（TARGET_MOD/AUTO_MOD_START/RELEASE_MODE，逐字）+ conf.lua（identity 存档隔离/窗口标题/图标）+ love.exe 拼 .love → DR-CH4-DOG-release.exe。产物对照表（mod.zip / win64 / .love），什么时候给哪种。

## 自定义图标

三步从易到难：window_icon.png（mod 根目录，运行时窗口图标+标题，引擎自动，独立包同样生效）→ icon.png（主菜单列表图标）→ exe 文件图标（icon.ico + icon.res 重编译，进阶，参考 wiki releasing-mods）；PNG 方形即可，素材按 EX02 规矩。

## 测试发布包

玩家视角冒烟（干净 Kristal 放 zip → 启动 → 主菜单 → 战斗 → 饶恕胜利 → 输一局看失败流程 → 退出查存档污染）；独立包额外检查（窗口标题/图标、debug 菜单没了、存档独立）。"交付前最后一分钟检查"（EX06 冒烟思路）。

## 整理作品说明

README 骨架（名称/运行方式/素材与许可）代码块；THIRD_PARTY.md 确认；发布 = 让陌生人能自己上手。

## GitHub 自动打包

从零讲"GitHub 能自动打包"：.github/ 里的流水线剧本——打上版本标签 → 自动跑打包脚本 → 自动传 GitHub Releases 页面；打标签两条路线（手动 git tag + push --tags；release-please 机器人合并 PR 自动打）；产物含 .love / win64 / mod.zip / SHA256SUMS，sha256sum -c SHA256SUMS 校验。

## 发布与回顾

第 03 集第一章清单全部打勾回顾（规划 → 战斗闭环 → 交付）；截图占位（Releases 页面）。

## 结尾

第一章收官（启动画面 → exe）；预告第二章 Tiled 地图；作业（建议语气）：把 mod.zip 装进干净 Kristal 跑全流程模拟玩家 / 改 name、subtitle 重新 build 看窗口标题变化。
