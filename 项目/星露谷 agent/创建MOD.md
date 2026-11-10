
# 创建一个C#Mod

> 读取游戏信息

## 开发Prompt

prompt 1：读取到基本的数据

```Plain
你现在是 StardewAgent 项目的 C# / SMAPI 开发工程师。

## 项目背景
我们正在开发 Stardew Valley Autonomous Agent，目标是让 AI 完全自主经营星露谷农场。架构为 C# SMAPI Mod + Python Agent。本次只实现 P1-01：Game Bridge 初始化和游戏状态读取。

## 已验证环境
- macOS 26.7.2，Apple Silicon ARM64
- Rosetta 2 已安装
- Stardew Valley 1.6.15 build 24356，x86_64
- 游戏目录：/Applications/Stardew Valley.app/Contents/MacOS
- SMAPI 4.5.2 已安装并验证能够正常加载
- .NET SDK 10.0.403 ARM64
- 游戏运行时 .NET 6.0.32（由游戏自身提供）
- Python 3.13.2 / uv 0.10.4

## 任务
1. 先检查当前 StardewAgent 仓库结构，不要覆盖已有文件。
2. 创建 game-bridge/StardewAgentBridge C# Class Library。
3. TargetFramework 必须为 net6.0。注意 .NET 10 SDK 的 dotnet new 模板可能不支持直接使用 -f net6.0，可以先生成默认类库再编辑 csproj。
4. 使用 NuGet 包 Pathoschild.Stardew.ModBuildConfig 4.4.0。
5. 如自动检测游戏路径失败，在 csproj 设置 GamePath 为 /Applications/Stardew Valley.app/Contents/MacOS，不要修改游戏原始程序集。
6. 添加有效的 SMAPI manifest.json：Name、Author、Version、Description、UniqueID、EntryDll、MinimumApiVersion、UpdateKeys。
7. 实现 ModEntry : StardewModdingAPI.Mod，并订阅 GameLaunched、SaveLoaded、DayStarted。
8. Entry() 只负责初始化与注册事件，不要过早读取尚未初始化的 Game1 状态。
9. GameLaunched 输出 Mod loaded successfully。
10. SaveLoaded 输出世界已加载、日期、季节、游戏时间、玩家地图、玩家 tile 坐标、体力。
11. DayStarted 输出新一天的日期和时间。
12. 读取游戏数据前做好空值与游戏状态检查，避免在主菜单访问玩家对象。
13. 本阶段只读，不移动角色、不修改世界或存档。
14. 添加 .gitignore 和 README，说明构建、安装、启动、日志检查与常见错误。

## 构建及验证
- 使用 dotnet restore、dotnet build。
- 编译前检查 SMAPI 4.5.2 中相关 API 的实际签名。
- 确认生成的程序集和 manifest.json 能部署到游戏 Mods/StardewAgentBridge。
- 编译失败时根据真实错误修复，不得编造测试通过的结果。
- 不修改或覆盖 Stardew Valley 的原始 DLL、启动器或其他 Mod。
- 不要安装多余的 SDK 或全局软件。

## 交付要求
报告新增及修改的文件、使用的命令、构建结果、实际部署位置，以及需要人工进入游戏验证的步骤。

最终成功标准：启动游戏后，SMAPI 正常加载 StardewAgentBridge，进入测试存档后输出真实玩家状态。
```

prompt 1.1 ：读取到数据之后要更新

```Plain
请检查并修复 StardewAgentBridge 当前只能输出游戏初始状态、无法持续跟踪变化的问题。

已验证：
- P1-01 Mod 成功加载。
- SaveLoaded 和 DayStarted 可以读取初始游戏状态。
- 初始时间为 600，位置 FarmHouse (9,9)，体力 270。
- 但游戏继续运行时，日志未体现时间与位置变化。

请在现有代码上增量修改，不要重建项目。

要求：
1. 保留 SaveLoaded 和 DayStarted。
2. 注册 GameLoop.TimeChanged，在游戏时间每变化10分钟时更新快照，并在调试模式输出时间变化。
3. 注册 GameLoop.OneSecondUpdateTicked，约每秒采集一次完整状态。
4. 最新状态必须保存在可访问的 GameStateSnapshot 中，而不能只是打印日志。
5. 采集字段至少包括日期、季节、时间、当前位置、地图名称、体力和最大体力。
6. GameStateObserver 必须在 SMAPI 游戏主线程读取 Game1，不能在后台线程直接读取游戏对象。
7. 只有 Context.IsWorldReady 为 true 时才读取游戏世界。
8. 注册 ReturnedToTitle 事件，返回主菜单后清除旧快照，标记 world_ready=false。
9. 避免每秒输出大量重复 JSON，只在调试模式下输出字段变化。
10. 后续 Python Controller 获取状态时，必须能够拿到最新快照，而不是进入存档时的初始快照。
11. 不实现 HTTP、动作执行或 LLM。
12. 保留现有功能，不覆盖其他 Mod 或游戏文件。

验收：
- 游戏时间从600变化为610、620时，内存快照同步变化。
- 人工移动角色后，快照坐标跟随变化。
- 体力变化后，快照体力跟随变化。
- 返回主菜单后不再暴露旧存档的状态。
- dotnet build 成功，SMAPI 正常加载。

请根据实际项目结构修改，并说明如何验证这些行为。
```

prompt 1.3：

```bash
请对 StardewAgentBridge 的 P1-02 状态感知模块进行完整扩展。

项目目标：开发完全自主经营星露谷物语的 Agent。当前 Mod 已经能读取日期、时间、位置和体力，但还缺少大量关键角色状态，而且部分状态没有随游戏变化持续更新。

运行环境：
- Stardew Valley 1.6.15 build 24356
- SMAPI 4.5.2
- .NET 6 Target Framework，.NET 10 SDK
- macOS Apple Silicon + Rosetta

本次仅开发状态观察层，不接入 Python、HTTP、LLM，不执行任何修改游戏世界的动作。

一、世界状态
读取年份、季节、日期、星期、游戏时间、天气、地图名称、是否暂停。星期使用游戏/SMAPI 提供的日期工具，不使用现实日期。

二、角色状态
读取 Tile 坐标、方向、当前体力、最大体力、生命值、最大生命值、当前技能等级。

三、背包状态
读取解锁容量、已使用格数、空格数以及所有已解锁格子的内容，保留空格子的索引。每个物品至少包含 slot_index、qualified_item_id、display_name、stack、quality、item_type。工具额外读取升级等级（若适用）。不要使用已废弃的 ParentSheetIndex 作为全局唯一 ID。

四、快捷道具栏
读取当前快捷栏对应的物品和顺序、选中索引、实际持有物品及其 ID。注意背包不等于快捷栏，切换快捷栏行后必须保持准确，不能假设始终为背包前12格。

五、装备
读取帽子、上衣、裤子、鞋子、左戒指、右戒指和 1.6 饰品槽。每个装备包含 qualified ID 和显示名称，空槽返回 null。区分装备槽与背包中的工具、武器。

六、经济
读取当前金币、游戏记录的累计收入，明确个人与团队口径。不得将当前金币、累计收入、当日出货金额混为一谈。不能准确读取的字段返回 null，并说明原因，不能编造数值。

七、任务
读取当前任务日志中的任务 ID、标题、描述、当前目标、完成状态、可获取的进度、期限和奖励。任务没有某些字段时使用 null。区分普通任务与特殊订单，不能将全部任务统一假设为普通任务。

八、状态更新
- SaveLoaded：初始化完整快照
- DayStarted：刷新新一天的状态
- TimeChanged：更新时间
- InventoryChanged：刷新背包
- OneSecondUpdateTicked 或等效机制：刷新动态角色状态，并定期校验其它状态
- ReturnedToTitle：清空旧状态
- 世界未加载时不得访问游戏状态

九、工程要求
1. 采用 GameStateSnapshot 与 WorldState、PlayerState、InventoryState、ToolbarState、EquipmentState、FinanceState、QuestState 等独立 DTO。
2. 仅在游戏主线程访问 Game1 和 Farmer 等游戏对象。
3. DTO 只包含可序列化的基础数据，不直接序列化游戏内部对象。
4. 使用 System.Text.Json，固定 snake_case 命名。
5. 维护 latestSnapshot、revision、world_ready。
6. 有数据变化时更新 revision。
7. 日志只输出变化摘要，不要每秒打印完整 JSON。
8. 为每类状态增加独立的观察器或采集方法。
9. 对不确定的 API 在当前已安装的游戏和 SMAPI 程序集中确认签名，不要根据旧版本猜测。
10. 不允许通过修改存档、游戏内存字段或原始 DLL 的方式完成读取。
11. 保留现有功能和项目文件，增量修改并构建测试。

验收要求：
- 游戏日期、星期、时间正确；
- 角色移动后位置更新；
- 选择不同工具后 selected_item 更新；
- 获得、消耗、丢弃物品后背包更新；
- 升级背包后容量正确；
- 更换装备后装备状态更新；
- 购买或销售后经济数据更新；
- 接受任务、推进任务、完成任务后任务状态更新；
- 返回主菜单后不暴露旧状态；
- dotnet build 成功，SMAPI 能正常加载。

完成后请报告每个字段使用的游戏 API、哪些字段已经实现、哪些暂时无法获得、哪些测试已经实际运行。禁止将未验证功能标记为通过。
```