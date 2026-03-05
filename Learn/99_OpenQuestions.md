> 目的：Phase 3（Engine Tick）已完成骨架梳理与证据落地；进入 Phase 4：以“倒推优先”的方式补齐 **Present → RenderFrame → Tick → 初始化点** 的最短证据链。
# 99_OpenQuestions
最后更新：2026-03-05

---

## 0. 已确认事实（权威：100_DoD）
- RunLoop 退出条件：只有 `m_currentState == nullptr` 才会退出；`PumpMessages=false` 触发的 `RequestExit(-66)` 只是标记，并不直接终止 RunLoop。
- 退出链路（已记录到 DoD）：`PumpMessages=false → RequestExit(-66) → RequestedExit 被 ProcessFrame 读取 → MainLoopSingleTick 逐层返回 false → Running 切到 Shutdown → TickStates 推进至 NextState 将 m_currentState 置空 → 退出 RunLoop`。
- D3D12/SwapChain 初始化证据（已记录到 DoD call stack）：
  - `GpuApi::InitDevice / CreateCommandQueue` 在 BaseInitialization 的 `OnEnterState` 链路上
  - `RenderViewport::CreateSwapChain` 在 `CRenderInterface::CreateViewport` 的链路上
- 线程与 JobSystem（已记录到 DoD）：Running 约 90 线程，存在 render thread；`job::Shutdown()` 被注释（需后续阶段再核对“是否确实为空实现/是否有替代 shutdown”）。

---

## 1. 当前阶段目标（Phase 4：第一帧渲染与 Present 倒推）
> 规则：本阶段只做倒推闭环，不扩散读文件；每次只推进 1~3 个函数/文件。

### 1.1 Present 的直接调用点（必须）【下一步就做这个】
目标：
- 找到 `Present()` / `Present1()` 的直接调用点（或最外层封装）。
- 抓到：源码路径+行号 + Call Stack + 命中线程（主线程/渲染线程）。

你要做的动作（限制 1~2 文件）：
- 从 DoD 已知点 `RenderViewport::CreateSwapChain()` 所在源码文件开始：
  1) 在该文件内搜 `Present(` / `Present1(` / `m_swapChain` 使用点
  2) 找到后对 `...Present...` 那行下断点
  3) 从程序启动开始跑，记录第一次命中证据（Call Stack 25 层 + Threads）

证据模板：
- Evidence-P1：`<file path>:<line>` + `...Present(...)` 表达式 + 线程 + Call Stack(25)

DoD（验收）：
- 能明确回答：Present 在哪个模块/函数被调用？发生在哪条线程？

---

### 1.2 第一帧 Present 的阶段归属（必须）
目标：
- 证明“第一帧 Present”发生在：
  - BaseInitialization/Splash 的 `OnEnterState` 阶段？
  - 还是 Running 的 `MainLoopSingleTick` 帧中？

你要做的动作：
- 仍使用 1.1 的 Present 断点：
  - 记录第一次命中时 call stack 是否出现 `...OnEnterState` / `...OnTick`
  - 或在断点处观察 `m_currentState` 的动态类型

DoD：
- 给出“第一帧 Present”的阶段结论，并能指向证据（栈/观察值）。

---

### 1.3 Present 上溯到帧骨架（必须）
目标：
- 把 Present 上溯到你 Phase 3 已整理的帧骨架节点：
  - `LoopSingleTick → Process(Main|Base)LoopFrame → GRender->Tick → ...Present...`

你要做的动作：
- 从 Evidence-P1 的 Call Stack 中提取是否出现：
  - `CBaseEngine::ProcessMainLoopFrame` / `CBaseEngine::ProcessBaseLoopFrame`
  - `CRenderInterface::Tick` / `GRender->Tick`
- 若缺失：说明 Present 可能在 render thread/其他线程路径中触发；暂不扩散读文件，先把“线程归属”证据补齐（进入 Phase 5）。

DoD：
- 产出一条“Present → 上游帧骨架”的最短链路（至少 5 个节点），每个节点都能落到符号名（最好含路径+行号）。

---

## 2. 进入下一阶段（Phase 5）的触发条件
当 1.1~1.3 完成后：
- 若 Present 不在主线程：开始 Phase 5（线程模型/同步点/Render thread 创建点）
- 若 Present 在主线程且能上溯到 Main/Base：继续在 Phase 4 补齐 “RenderFrame 的最外层函数 + D3D12 backend 初始化点（Device/Queue/Fence/SwapChain）” 的完整地图。