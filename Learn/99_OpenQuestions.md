> 目的：你已经把 launcher 的入口、RunLoop 与 StateMachine 骨架整理出来了；下一步要把 **“每帧到底执行了什么（GEngine Tick）”**、**“第一帧渲染/Present 在哪里发生”**、以及 **“退出与 Shutdown 如何让 Run() 结束”** 做成可验证的证据闭环。

# 99_OpenQuestions
最后更新：2026-03-04

---

## 0. 已整理出的骨架（来自现有笔记的“确认项”）
> 这一节是“本阶段默认事实”。若后续被真实源码断点推翻，请同步更新 `Learn/100_DoD.md`。

### 0.1 主循环（GameAppInstance::Run）
- 循环结构：每轮先 `PumpMessages()`，再 `TickStates()`；循环条件是 `while ( m_currentState )`
- 若 `PumpMessages()` 返回 false：会记录 error log，并执行 `GEngine->RequestExit(-66)`（注意：此处只是 RequestExit，不等价于“立刻结束 RunLoop”，退出仍取决于状态机是否把 `m_currentState` 推到 null）
- 关联笔记：
  - `Learn/01_Bootstrap/03_GameAppInstance/99_Run.md`

### 0.1.1 平台消息泵（PlatformPumpMessages）【你刚补充的确认项】
- `PlatformPumpMessages()` 内部仅做 Win32 `PeekMessageW` 循环：
  - 遇到 `WM_QUIT` 返回 false
  - 否则 `TranslateMessage` + `DispatchMessageW`，最终返回 true
- `AppInstance::PumpMessages()`（或同等封装）只是转发到 `platform::PlatformPumpMessages()`，不包含额外逻辑
- 关联笔记：
  - `Learn/03_PumpMessages/01_PumpMessages.md`

### 0.2 状态机推进（GameAppInstance::TickStates）
- 状态上下文：`EnterState -> Tick -> ExitState -> NextState`
- `OnEnterState()` 返回 true 才进入 Tick，否则直接 Exit
- `OnTick()` 返回 true 表示当前 State 的 Tick 完成，需要进入 Exit
- Exit 后进入 NextState：`m_currentState = m_nextState; m_nextState = nullptr;`
- 当 `m_nextState == nullptr` 且完成一次 `NextState` 后，`m_currentState` 会变为 null，从而退出 Run 循环（此点仍要求你在真实源码里用断点证据确认）
- 关联笔记：
  - `Learn/02_TickStates/01_TickStates.md`

### 0.3 Running 状态（GameAppRunningState::OnTick）
- 每帧调用 `GEngine->MainLoopSingleTick()`
- 若返回 false：若尚未 RequestedExit，则 `RequestExit(-42)`，并请求状态切换到 `GameAppState_Shutdown`
- 关联笔记：
  - `Learn/02_TickStates/OnTick/03_Running.md`

### 0.4 Initialization / BaseInitialization 的 EngineState 驱动
- BaseInitialization：
  - `BES_InitializingRegularSystems` → 切到 `GameAppState_Initialization`
  - `BES_Initialized` → 可直接切到 Running
  - `BES_InitializingRootSystems` 会 `YieldCurrentThread`
- Initialization：
  - `BES_InitializingRegularSystems` 会跑 `GEngine->BaseLoopSingleTick()`（“简化帧”）
  - 直到 `BES_Initialized` 后切 Running
- 关联笔记：
  - `Learn/02_TickStates/OnTick/01_BaseInitializationState.md`
  - `Learn/02_TickStates/OnTick/02_InitializationState.md`

### 0.5 Engine Tick 的最外层骨架（Base vs Main）【你已整理进笔记】
- `MainLoopSingleTick()`：`LoopSingleTick( &CBaseEngine::ProcessMainLoopFrame )`
- `BaseLoopSingleTick()`：`LoopSingleTick( &CBaseEngine::ProcessBaseLoopFrame )`
- `LoopSingleTick()`：时间步长/统计/帧内存回收/再调用 singleTickFun（Main/Base frame）
- 关联笔记：
  - `Learn/01_Bootstrap/04_Loop/02_MainLoopSingleTick.md`
  - `Learn/01_Bootstrap/04_Loop/01_BaseLoopSingleTick.md`
  - `Learn/01_Bootstrap/04_Loop/03_LoopSingleTick.md`
  - `Learn/01_Bootstrap/04_Loop/05_ProcessMainLoopFrame.md`
  - `Learn/01_Bootstrap/04_Loop/04_ProcessBaseLoopFrame.md`

### 0.6 Shutdown 状态
- `ShutdownEngine()` → `gameApp.SetReturnValue()` → `gameApp.Shutdown()`
- 然后关闭 http/logging/io/network（注意 `job::Shutdown()` 被注释掉：需要你确认原因与是否存在替代 shutdown 路径）
- 关联笔记：
  - `Learn/02_TickStates/OnTick/04_Shutdown.md`

---

## 1. 本阶段要达成的“新闭环”（你接下来要做的事）
> 仍坚持“小步闭环”：每次只推进 1~3 个函数/文件，用断点+调用栈收证据。

### 1.1 闭环 A：PumpMessages → RequestExit → MainLoop 返回 false 的链路（必须）
现状（已确认）：
- `PlatformPumpMessages` 的 false 仅来自 `WM_QUIT`
- `Run()` 在 `PumpMessages()==false` 时调用 `GEngine->RequestExit(-66)`

剩余未闭环点（下一步要证明）：
- `RequestExit(code)` **如何**影响：
  1) `GEngine->MainLoopSingleTick()` 的返回值（是否因 flag 直接返回 false？）
  2) 状态机何时切到 Shutdown，并最终让 `m_currentState` 变 null

证据采集（建议只做 2 个断点就够）：
- [ ] 定位 `GEngine->RequestExit(...)` 的实现文件路径+行号，并在函数入口下断点
- [ ] 定位 `MainLoopSingleTick()`（或其内部被检查 exit flag 的点）：
  - 抓一帧 Call Stack（20 层左右）
  - 观察：exit flag/RequestedExit 相关字段在何处被读取并决定返回值

DoD（验收）：
- 你能写清楚：**WM_QUIT → PumpMessages=false → RequestExit(-66) →（何处读 flag）→ MainLoopSingleTick=false → Running 请求切 Shutdown → TickStates 退出 RunLoop** 的最短链路（每一跳都能落到“文件路径+行号”）。

---

### 1.2 闭环 B：MainLoop vs BaseLoop 的“差异清单”与边界（必须）
现状（已整理进笔记，但还缺“真实源码路径+行号”的落地）：
- `ProcessMainLoopFrame` / `ProcessBaseLoopFrame` 的函数级骨架已经记录
- 关键差异（你笔记里已经可见）包括但不限于：
  - Main 多：ResourceLoader Update、ink BeginFrame/Tick、ProcessFrame、AsyncMaterialCompiler Tick、InGameConfig Update 等
  - Base 多：ink SimpleTick、ProcessBaseFrame（简化帧）

下一步（只做“落地证据”，别扩散读文件）：
- [ ] 在 VS 里对下列函数做“转到定义”，记录真实源码：文件路径 + 行号
  - `CBaseEngine::ProcessMainLoopFrame`
  - `CBaseEngine::ProcessBaseLoopFrame`
  - `CBaseEngine::ProcessFrame`
  - `CBaseEngine::ProcessBaseFrame`
- [ ] 每个函数抓 1 次断点命中 Call Stack（20 层），用于证明它属于 Main/Base 哪条链路

DoD：
- 你能给出一张“差异清单”并注明每条差异对应的源码位置（路径+行号）。

---

### 1.3 闭环 C：第一帧渲染 / Present 倒推（强烈建议：下一步优先做这个）
目标：
- 找到 Present 的**直接调用点**（swapchain present 或最外层封装）
- 把 Present **上溯到**：`GRender->Tick()` → `Process(Main/Base)LoopFrame` → `LoopSingleTick` → `Main/BaseLoopSingleTick`
- 同时确认 Present 发生在线程：主线程 / render thread / job thread（以 Threads 窗口为证）

建议倒推法（一次只推进 1~2 个文件）：
- [ ] 在 VS 全工程搜索（只记录命中“最靠近 present”的 1~3 个点的文件路径，不要开始读一堆文件）：
  - `Present(` / `Present1(` / `IDXGISwapChain`
  - `CreateSwapChain` / `CreateSwapChainForHwnd`
- [ ] 在最靠近 Present 的**调用点**下断点，启动后抓 1 帧证据：
  - Call Stack（25 层）
  - Threads 窗口：命中线程是谁
  - 栈里是否出现：`GRender->Tick` / `ProcessMainLoopFrame` / `ProcessBaseLoopFrame`

DoD：
- 你能把 **Present 上溯到 MainLoop/BaseLoop 的链路**说清楚，并能回答：
  - 第一帧 Present 更可能发生在 Initialization（BaseLoop）还是 Running（MainLoop）？（以断点命中时的 State 为证）

---

### 1.4 闭环 D：Shutdown 如何让 Run() 结束（必须）
目标：
- 证明 `m_currentState` 最终如何变成 null（从而跳出 `do { ... } while (m_currentState)`）

证据采集（推荐只推进 1 个函数入口 + 1 个分支断点）：
- [ ] 定位 `RequestNextStateTransition()` 的实现：它如何设置 `m_nextState`
- [ ] 在 `TickStates()` 的 `StateContext_NextState` 分支处下断点：
  - 观察：`m_nextState == nullptr` 时是否把 `m_currentState` 置空
  - 记录：最后一次循环中 `m_currentContext` 的序列（Enter/Tick/Exit/NextState）

DoD：
- 你能用断点观察结果写清楚：**退出 RunLoop 的唯一条件是什么**，以及它是如何被 Shutdown 路径触发的。

---

### 1.5 线程与 JobSystem（可选但很关键）
你已在 BaseInitialization Enter 里看到 `job::Initialize()`（以及可配置线程数）。下一步建议：
- [ ] 记录 job 线程池默认线程数计算逻辑（CPU 核数？常量？配置？）
- [ ] VS Threads 窗口记录：Running 状态下有多少条线程、是否存在独立 render thread
- [ ] 查找并记录 `job::Shutdown()` 被注释的原因（是否有替代 shutdown、或由更高层统一释放）

---

## 2. Open Questions（本阶段卡点清单）
### 2.1 RunLoop / Exit
- Q1：`PumpMessages()` 返回 false 的原因有哪些？
  - 已确认：平台层 `PlatformPumpMessages` 的 false 来自 `WM_QUIT`
  - 待确认：是否存在其他平台封装层的失败原因（如果有，需要真实源码证明）
- Q2：`RequestExit(code)` 到底如何影响 `MainLoopSingleTick()`？（是否仅设置 flag，flag 在哪里被读取）
- Q3：`m_currentState` 变 null 的具体代码路径是什么？（必须定位到源码）

### 2.2 Engine Tick
- Q4：`BaseLoopSingleTick()` 的“简化帧”包含哪些子系统？（已在 `ProcessBaseLoopFrame` 可见，仍需源码路径落地）
- Q5：`MainLoopSingleTick()` 相对 BaseLoop 多了哪些环节？（同上）
- Q6：`GEngine->GetState()` 的状态推进（RootSystems -> RegularSystems -> Initialized）在哪里发生？

### 2.3 Render / Present
- Q7：Present 的最外层封装函数叫什么？在哪个模块？
- Q8：第一帧渲染发生在 Initialization（BaseLoop）还是 Running（MainLoop）？
- Q9：D3D12 设备/交换链创建在哪个阶段调用？

### 2.4 Shutdown
- Q10：`ShutdownEngine()` 的核心释放顺序是什么？
- Q11：为何 `job::Shutdown()` 被注释？是否会造成资源泄露或死锁风险？
- Q12：`io::Shutdown()` / `logging::Shutdown()` 的顺序是否有强依赖？

---

## 3. Evidence 模板（你提交学习记录时按这个填）
> 每条证据尽量包含 “文件路径 + 行号 + 断点/调用栈”。

- [ ] `GameAppInstance::Run`：文件路径 + 行号 + Call Stack（需包含 `PumpMessages=false → RequestExit(-66)`）
- [ ] `PlatformPumpMessages`：文件路径 + 行号 + Call Stack（true/false 两条）
- [ ] `GameAppInstance::TickStates`：文件路径 + 行号 + 观察到的 context 序列
- [ ] `RequestNextStateTransition`：文件路径 + 行号 + 断点观察到的 m_nextState 设置逻辑
- [ ] `GEngine::RequestExit`：文件路径 + 行号 + Call Stack（至少 1 次）
- [ ] `GEngine::MainLoopSingleTick`：文件路径 + 行号 + Call Stack（1 帧）
- [ ] `GEngine::BaseLoopSingleTick`：文件路径 + 行号 + Call Stack（1 帧）
- [ ] `CBaseEngine::ProcessMainLoopFrame`：文件路径 + 行号 + Call Stack（1 帧）
- [ ] `CBaseEngine::ProcessBaseLoopFrame`：文件路径 + 行号 + Call Stack（1 帧）
- [ ] Present 断点：文件路径 + 行号 + Call Stack（1 帧）+ Threads 截图/文字记录
- [ ] Shutdown 路径：`ShutdownEngine` / `gameApp.Shutdown` / `StateContext_NextState` 的关键证据

---

## 4. 我（AI）拿到这些证据后会输出什么
当 1.1~1.4 的证据齐全后，我会输出：
1) “一帧执行地图（函数级）”：MainLoop vs BaseLoop 的差异版
2) “退出与关机地图”：从 WM_QUIT/RequestExit 到 RunLoop 结束的最短链路
3) “渲染倒推路线”：Present -> RenderTick -> Tick -> 初始化位置（含线程归属）
4) 下一步只让你再深入 1~3 个文件（严格最短路径推进）> 目的：你已经把 launcher 的入口、RunLoop 与 StateMachine 骨架整理出来了；下一步要把 **“每帧到底执行了什么（GEngine Tick）”**、**“第一帧渲染/Present 在哪里发生”**、以及 **“退出与 Shutdown 如何让 Run() 结束”** 做成可验证的证据闭环。

# 99_OpenQuestions

最后更新：2026-03-03

---

## 0. 已整理出的骨架（来自现有笔记的“确认项”）
> 这一节是“本阶段默认事实”，后续若被证据推翻，请更新 `100_DoD.md`。

### 0.1 主循环（GameAppInstance::Run）
- 循环结构：每轮先 `PumpMessages()`，再 `TickStates()`；循环条件是 `while ( m_currentState )`  
- `PumpMessages()` 失败会触发 `GEngine->RequestExit(-66)`（你后续需要确认这是否足以导致 MainLoopSingleTick 返回 false）

关联笔记：
- `Learn/01_Bootstrap/03_GameAppInstance/99_Run.md`

### 0.2 状态机推进（GameAppInstance::TickStates）
- 状态上下文：`EnterState -> Tick -> ExitState -> NextState`
- `OnEnterState()` 返回 true 才进入 Tick，否则直接 Exit
- `OnTick()` 返回 true 表示当前 State 的 Tick 完成，需要进入 Exit
- Exit 后进入 NextState：`m_currentState = m_nextState; m_nextState = nullptr;`
- 当 `m_nextState == nullptr` 且完成一次 `NextState` 后，`m_currentState` 会变为 null，从而退出 Run 循环（这点需要你在真实源码里用断点证据确认）

关联笔记：
- `Learn/02_TickStates/01_TickStates.md`

### 0.3 Running 状态（GameAppRunningState::OnTick）
- 每帧调用 `GEngine->MainLoopSingleTick()`
- 若返回 false：若尚未 RequestedExit，则 `RequestExit(-42)`，并请求状态切换到 `GameAppState_Shutdown`

关联笔记：
- `Learn/02_TickStates/OnTick/03_Running.md`

### 0.4 Initialization / BaseInitialization 的 EngineState 驱动
- BaseInitialization：根据 `GEngine->GetState()` 在 `BES_InitializingRegularSystems` 时切到 `GameAppState_Initialization`；在 `BES_Initialized` 时可直接切到 Running  
  - `BES_InitializingRootSystems` 会 `YieldCurrentThread`
- Initialization：`BES_InitializingRegularSystems` 会跑 `GEngine->BaseLoopSingleTick()`（“简化帧”），直到 `BES_Initialized` 后切 Running

关联笔记：
- `Learn/02_TickStates/OnTick/01_BaseInitializationState.md`
- `Learn/02_TickStates/OnTick/02_InitializationState.md`

### 0.5 Shutdown 状态
- `ShutdownEngine()` → `gameApp.SetReturnValue()` → `gameApp.Shutdown()`  
- 然后关闭 http/logging/io/network（注意 `job::Shutdown()` 被注释掉：这需要你确认原因与是否存在替代 shutdown 路径）

关联笔记：
- `Learn/02_TickStates/OnTick/04_Shutdown.md`

---

## 1. 本阶段要达成的“新闭环”（你接下来要做的事）

### 1.1 闭环 A：PumpMessages 的真实行为与退出触发条件（必须）
目标：回答两件事（必须能用断点/源码行号证明）：
1) `PumpMessages()` 什么时候会返回 false？
2) `GEngine->RequestExit(code)` 如何影响 `MainLoopSingleTick()` 的返回值与退出路径？

你要做的证据采集：
- [ ] 在真实源码中定位 `GameAppInstance::PumpMessages()` 的文件路径与行号
- [ ] 断点：`PumpMessages()` return 之前（true/false 两条路径都要）
- [ ] 断点命中时的 Call Stack（10~20 层）
- [ ] 记录：`PumpMessages()` 内部是否有 Win32 message pump（GetMessage/PeekMessage/DispatchMessage）或引擎封装

DoD（验收）：
- 你能写清楚：**退出是“消息泵失败”触发的，还是“MainLoopSingleTick 返回 false”触发的**，以及两者的先后关系。

---

### 1.2 闭环 B：一帧 Tick 的“最外层骨架”在引擎里是什么（必须）
目标：把 `MainLoopSingleTick()` 与 `BaseLoopSingleTick()` 各自的职责拆清楚，并回答：
- `BaseLoopSingleTick()` 里到底 tick 了哪些子系统（至少列出 Input/Render/Audio/Job/World 等中你能证到的部分）
- `MainLoopSingleTick()` 在 `BaseLoopSingleTick()` 基础上额外做了什么（世界更新？脚本？游戏逻辑？）

你要做的证据采集：
- [ ] 定位并记录源码路径：
  - `GEngine->MainLoopSingleTick()` 实现在哪里
  - `GEngine->BaseLoopSingleTick()` 实现在哪里
  - `GEngine->GetState()` 的状态推进在哪里发生（谁把 state 从 RootSystems 推到 RegularSystems/Initialized）
- [ ] 在 `MainLoopSingleTick()` 的函数体开头下断点，抓 1 帧 Call Stack（20 层左右）
- [ ] 在 `BaseLoopSingleTick()` 的函数体开头下断点，抓 1 帧 Call Stack（20 层左右）
- [ ] 若两者会互相调用：记录“调用边”（谁调用谁、在第几层）

DoD：
- 你能画出并解释一帧的 5~10 个关键步骤（函数级），并能指出每一步对应源码文件路径。

---

### 1.3 闭环 C：第一帧渲染/Present 在哪里发生（强烈建议）
目标：从 “Tick -> Render -> Present” 建立最短证据链：
- Present 的直接调用点（swapchain present 或封装函数）
- RenderFrame 的最外层函数
- D3D12 backend 初始化发生在哪一段（BaseInitialization Enter / Engine init / first tick）

建议倒推法（更快）：
- [ ] 在 VS “全工程搜索”关键词（命中点记录文件路径）：
  - `Present(` / `IDXGISwapChain` / `CreateSwapChain`
  - `D3D12CreateDevice` / `CreateCommandQueue` / `CreateFence`
- [ ] 在最靠近 Present 的函数下断点，启动后抓 1 帧 Call Stack

DoD：
- 你能把 **Present 上溯到 MainLoopSingleTick/BaseLoopSingleTick 的链路**说清楚。

---

### 1.4 闭环 D：Shutdown 如何让 Run() 结束（必须）
目标：证明 `m_currentState` 最终如何变成 null（从而跳出 `do { ... } while (m_currentState)`）。

你要做的证据采集：
- [ ] 定位 `RequestNextStateTransition()` 的实现：它如何设置 `m_nextState`
- [ ] 定位 shutdown 状态是否实现 `OnExitState()`（或继承默认实现）；Exit 后是否会请求 nextState
- [ ] 在 `TickStates()` 的 `StateContext_NextState` 分支处下断点：
  - 观察：`m_nextState == nullptr` 时是否把 `m_currentState` 置空
  - 观察：最后一次循环中 `m_currentContext` 的序列

DoD：
- 你能用断点观察结果写清楚：**退出 RunLoop 的唯一条件是什么**，以及它是如何被 Shutdown 路径触发的。

---

### 1.5 线程与 JobSystem（可选但很关键）
你已在 BaseInitialization Enter 里看到 `job::Initialize()`（以及可配置线程数）。下一步建议：
- [ ] 记录 job 线程池默认线程数计算逻辑（CPU 核数？常量？配置？）
- [ ] VS Threads 窗口记录：Running 状态下有多少条线程、是否存在独立 render thread
- [ ] 查找并记录 `job::Shutdown()` 被注释的原因（是否有替代 shutdown、或由更高层统一释放）

---

## 2. Open Questions（本阶段卡点清单）

### 2.1 RunLoop / Exit
- Q1：`PumpMessages()` 返回 false 的原因有哪些？是否会主动触发退出？
- Q2：`RequestExit(code)` 到底如何影响 `MainLoopSingleTick()`？（是否仅设置 flag）
- Q3：`m_currentState` 变 null 的具体代码路径是什么？（必须定位到源码）

### 2.2 Engine Tick
- Q4：`BaseLoopSingleTick()` 的“简化帧”包含哪些子系统？
- Q5：`MainLoopSingleTick()` 相对 BaseLoop 多了哪些环节？
- Q6：`GEngine->GetState()` 的状态推进（RootSystems -> RegularSystems -> Initialized）在哪里发生？

### 2.3 Render / Present
- Q7：Present 的最外层封装函数叫什么？在哪个模块？
- Q8：第一帧渲染发生在 Initialization 阶段（BaseLoop）还是 Running 阶段（MainLoop）？
- Q9：D3D12 设备/交换链创建在哪个阶段调用？

### 2.4 Shutdown
- Q10：`ShutdownEngine()` 的核心释放顺序是什么？
- Q11：为何 `job::Shutdown()` 被注释？是否会造成资源泄露或死锁风险？
- Q12：`io::Shutdown()` / `logging::Shutdown()` 的顺序是否有强依赖？

---

## 3. Evidence 模板（你提交学习记录时按这个填）
> 每条证据尽量包含 “文件路径 + 行号 + 断点/调用栈”。

- [ ] `GameAppInstance::Run`：文件路径 + 行号 + Call Stack
- [ ] `GameAppInstance::PumpMessages`：文件路径 + 行号 + Call Stack（true/false 两条）
- [ ] `GameAppInstance::TickStates`：文件路径 + 行号 + 观察到的 context 序列
- [ ] `GEngine::MainLoopSingleTick`：文件路径 + 行号 + Call Stack（1 帧）
- [ ] `GEngine::BaseLoopSingleTick`：文件路径 + 行号 + Call Stack（1 帧）
- [ ] Present 断点：文件路径 + 行号 + Call Stack（1 帧）
- [ ] Shutdown 路径：`ShutdownEngine` / `gameApp.Shutdown` / `RequestNextStateTransition` 的关键证据

---

## 4. 我（AI）拿到这些证据后会输出什么
当 1.1~1.4 的证据齐全后，我会输出：
1) “一帧执行地图（函数级）”：MainLoop vs BaseLoop 的差异版
2) “退出与关机地图”：从 RequestExit 到 RunLoop 结束的最短链路
3) “渲染倒推路线”：Present -> RenderFrame -> Tick -> 初始化位置
4) 下一步只让你再深入 1~3 个文件（严格最短路径推进）