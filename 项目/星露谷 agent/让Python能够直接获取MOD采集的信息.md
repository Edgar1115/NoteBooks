
> 让 Python 能够直接获取 C# Mod 采集的最新游戏状态，而不是从 SMAPI 控制台读取日志。

prompt 1 : 

```bash
继续开发 StardewAgent，执行 P1-03：Game Bridge 与 Python Agent Controller 的本地 HTTP 通信。

已有成果：
- P1-01：StardewAgentBridge Mod 已成功加载
- P1-02：world、player、inventory、toolbar、equipment、finance、quests 七个模块已经实现
- 游戏版本 Stardew Valley 1.6.15，SMAPI 4.5.2
- C# 项目目标框架 net6.0
- macOS Apple Silicon，游戏通过 Rosetta x86_64 运行

目标：
实现 C# Game Bridge 主动向 Python FastAPI 推送游戏状态，Python 验证并保存最新快照。

一、检查现有代码
1. 首先检查真实项目结构与七个模块的数据模型。
2. 保留现有字段与序列化规则，不要重新发明一套不兼容的 Snapshot。
3. 检查状态更新机制是否已经正常实现。
4. 不得覆盖现有 Mod 或重建项目。

二、Python Controller
1. 如果尚不存在 Python 工程，则在 agent/ 中使用 uv 创建。
2. 使用 FastAPI、Uvicorn、Pydantic。
3. 实现 GET /health。
4. 实现 POST /v1/observations。
5. 实现 GET /v1/observations/latest。
6. 定义与 C# Snapshot 完全一致的 Pydantic Schema。
7. 验证 schema_version、session_id、sequence 和状态内容。
8. 拒绝同一 session_id 下旧 sequence 覆盖新数据。
9. 为不同游戏会话正确重置当前状态。
10. 存储最后接收时间，提供状态新鲜度判断。游戏退出、断开通信或长时间无数据时不能将旧状态标为实时。
11. 暂时只保存在内存中，不需要 SQLite。

三、C# Transport
1. 新增独立 Transport 模块。
2. 使用 System.Net.Http.HttpClient。
3. 通过现有 Snapshot DTO 的 JSON 序列化结果发送状态。
4. 默认发送地址 http://127.0.0.1:8765。
5. 网络请求必须异步执行，不能阻塞 SMAPI 游戏主线程。
6. 不允许后台线程读取 Game1 或访问其他游戏内部对象。
7. 主线程先采集 DTO 并完成不可变快照或序列化，再交给后台发送。
8. 默认约每现实秒推送一次。
9. 使用单发送者、有限缓冲或最新状态覆盖策略，避免网络故障时无限堆积请求。
10. 当 Python 服务关闭时，记录限频警告，游戏继续正常运行。
11. 实现连接恢复、请求超时与资源释放。
12. 退出存档时向 Python 传递 world_ready=false，并清理本地会话状态；同时使用心跳超时机制处理无法送达的情况。
13. 网络配置默认仅限 127.0.0.1，不暴露公网。

四、测试
1. Python 服务可以独立启动。
2. GET /health 返回正常状态。
3. 使用模拟 JSON 测试 Pydantic 校验。
4. C# 编译成功并正常加载到 SMAPI。
5. 游戏加载后 Python 能接收到真实 Snapshot。
6. 游戏时间、位置、背包、工具选择、现金变化能够传输到 Python。
7. Python 关闭后游戏仍能正常运行。
8. Python 重新启动后，C# 能重新连接。
9. 返回主菜单后不再暴露旧存档的实时状态。
10. 网络乱序、重复快照、无效 Schema 不得污染最新状态。

五、限制
- 暂时不实现游戏动作控制。
- 暂时不接入 LLM。
- 暂时不实现 AI 规划。
- 不修改游戏存档或原始程序集。
- 不在后台线程直接调用游戏 API。
- 优先复用已经完成的七个观察模块。

完成后：
报告修改文件、接口文档、运行命令、构建结果、测试结果和手动验收步骤。对于尚未实际测试的内容明确标记未验证。
```

prompt 2 :

```Plain
继续开发 StardewAgent 项目，执行 P1-04：Action Executor（游戏动作执行器）。

项目目标：
开发能够完全自主经营 Stardew Valley 的 AI Agent。当前已经完成 P1-01 C# Mod 初始化、P1-02 七个游戏状态观察模块、P1-03 Python FastAPI 与 C# Bridge 的 HTTP 通信，且 P1-03 已通过实际验收。

运行环境：
- macOS Apple Silicon，游戏进程运行于 Rosetta x86_64
- Stardew Valley 1.6.15 build 24356
- SMAPI 4.5.2
- .NET SDK 10.0.403，Mod TargetFramework net6.0
- Python + FastAPI + Pydantic + uv

目标：
让 Python Controller 能够下达基础动作，由 C# SMAPI Mod 实际执行，并返回可靠的动作执行状态与结果。

一、检查现有工程
1. 先阅读项目结构、Snapshot 模型、session_id、sequence、现有 HTTP 通信接口。
2. 复用 P1-03 的 HTTP 架构及会话机制。
3. 保留 P1-01 至 P1-03 的全部功能。
4. 不重建项目，不修改游戏原始程序集。

二、Python Action API
新增或扩展以下接口，注意与现有 API 保持命名一致：
- POST /v1/actions：提交动作，返回 action_id
- POST /v1/actions/claim：供 C# 获取下一条待执行动作，并建立有限时长的领取租约
- POST /v1/actions/{action_id}/status：C# 上报动作执行状态和结果
- GET /v1/actions/{action_id}：查询动作状态
- POST /v1/control/stop：设置高优先级停止请求

要求：
- 绑定当前 session_id。
- action_id 全局唯一。
- 同一 action_id 幂等，重复提交不能重复执行。
- 动作领取、租约和状态更新必须具有明确的一致性规则。
- 先限制单个运行中动作，不实现复杂并发执行。
- 有界队列，超出容量时明确拒绝。
- 不允许旧会话或已过期的动作在新存档执行。
- Python 服务重启后不能无条件重放此前已经领取的动作。

三、C# Action Transport
1. 在现有 Transport 基础上增加动作轮询。
2. 使用后台 HTTP 请求领取动作，不阻塞 SMAPI 主线程。
3. 仅在游戏主线程执行任何游戏相关操作。
4. 使用线程安全队列或等效机制在后台通信与主线程之间交接动作。
5. 网络失败时不能让动作队列无限积压。
6. 领取动作后回传 RUNNING、SUCCEEDED、FAILED、CANCELLED、REJECTED、TIMED_OUT 等状态。
7. 动作结果至少包含 action_id、session_id、开始与结束时间、执行 Tick 数、开始与结束位置、失败原因。
8. 维护心跳或租约过期机制，服务断联时停止当前自动操作。

四、Action Executor
本次优先实现：
- move_direction：up/down/left/right，指定持续 Tick 数。
- stop：停止正在执行的动作。

可在基础动作稳定后实现：
- select_slot：切换当前选择的背包槽位。
- face_direction：改变面向。
- use_tool：使用已装备工具。
- interact：与面前的对象交互。

不要在第一版实现 move_to 或自动寻路。

五、SMAPI 输入控制
1. 使用 SMAPI 4.5.2 中已存在的 IInputHelper.Press(SButton)。
2. 在适当的 GameLoop 更新事件中执行输入，不要在后台线程调用 Helper.Input.Press。
3. 按住按键时每个相关 Tick 重新调用 Press，达到时长后停止调用。
4. 不要直接使用 PyAutoGUI、macOS 辅助功能或全局键盘模拟。
5. 方向输入应尊重游戏中的实际键位映射，不能永久假设是 WASD。
6. 初期限制每次移动为 1 到 60 Tick，防止无限移动。
7. 处理角色移动被障碍物阻挡的情况，不能因为指令执行完就认为抵达目标。

六、安全控制
1. 默认自动控制为关闭状态，必须由用户明确开启。
2. 增加可配置的紧急停止快捷键，例如 F8。
3. 紧急停止直接在游戏主线程生效，不依赖 Python HTTP 服务。
4. 用户选择手动接管时，暂停自动执行并清空待执行动作。
5. 在主菜单、加载画面、游戏未就绪、对话或菜单界面中拒绝不合适的动作。
6. 世界会话发生变化后清除旧会话动作。
7. stop 必须优先于普通动作。
8. 达到最大 Tick 或实际时间限制时强制结束动作。
9. 不得意外修改、删除或覆盖用户存档。
10. 暂时仅在单人测试存档中启用自动控制。

七、测试
- Python API Schema 校验、队列状态机及幂等测试。
- C# Action Executor 逻辑测试。
- SMAPI 构建成功。
- Python 发送 move_direction 后游戏角色实际移动。
- 角色位置变化能够通过现有 Snapshot 观察。
- stop 能中断移动。
- F8 能在 Python 失联时紧急停止。
- 游戏菜单打开时不能继续乱移动。
- 在障碍物前不会无限移动。
- 返回主菜单后旧动作失效。
- Python 重启后不会自动重放旧动作。
- 动作执行结果可通过 GET /v1/actions/{action_id} 查询。

八、交付
请报告修改文件、API 定义、动作状态机、运行命令、测试结果与人工验收步骤。
实际未测试的功能必须标明未验证。

不要接入 LLM，不实现 A*、农场经营或自动规划。这些属于后续阶段。
```