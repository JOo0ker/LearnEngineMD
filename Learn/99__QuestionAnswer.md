1.1 `Present`在`render`线程/模块中的RenderViewport::Present函数调用
1.2 `Present`时，主线程的调用堆栈如下：
```
 	redJobs2.dll!job::prv::helper::PumpWindowsMessagesForDXGI() 行 1102	C++
 	redJobs2.dll!job::prv::helper::MessagePumper::Update() 行 1175	C++
 	redJobs2.dll!job::prv::Dispatcher::FlushCounter(const job::prv::CounterEntry * counterEntry, job::Priority processPriority, int timeoutMillseconds, bool processLarge) 行 1246	C++
 	redJobs2.dll!job::FlushCounterOnProcessFrame(const job::Counter & counter) 行 56	C++
 	gameFramework.dll!gsm::State_SessionStreamingAware::ProcessTick(const gsm::TickContext & context, gsm::StateTickFlagsContext & flags, gsm::TickControl & control) 行 139	C++
 	gameFramework.dll!gsm::State_SessionActive::ProcessTick(const gsm::TickContext & context, gsm::StateTickFlagsContext & flags, gsm::TickControl & control) 行 264	C++
 	gameFramework.dll!gsm::StateMachine::ProcessTick(float timeDelta, const tick::Info & tickInfo) 行 323	C++
 	gameFramework.dll!gsm::StateMachine::ProcessFrame(const gsm::Frame & frame) 行 291	C++
 	gameFramework.dll!gsm::CGameFramework::ProcessFrame(const gsm::Frame & frame) 行 234	C++
 	gameFramework.dll!CGameEngine::ProcessFrame(const float scaledEngineTimeDelta, const tick::Info & tickInfo, input::IInputContext & input) 行 974	C++
 	engine.dll!CBaseEngine::ProcessMainLoopFrame(const float timeDelta) 行 928	C++
 	engine.dll!CBaseEngine::LoopSingleTick(bool(CBaseEngine::*)(const float) singleTickFun) 行 660	C++
 	engine.dll!CBaseEngine::MainLoopSingleTick() 行 684	C++
 	engine.dll!red::GameAppRunningState::OnTick(red::GameAppInstance & gameApp) 行 32	C++
 	engine.dll!red::GameAppInstance::TickStates() 行 225	C++
 	engine.dll!red::GameAppInstance::Run() 行 113	C++
 	launcher.exe!`anonymous namespace'::mainRedGuarded() 行 80	C++
 	launcher.exe!mainRed() 行 94	C++
 	launcher.exe!wWinMain(HINSTANCE__ * hInstance, HINSTANCE__ * hPrevInstance, wchar_t * commandLine, int nCmdShow) 行 17	C++

```
1.3 具体链路调用如下，最终调用的是`IDXGISwapChain::Present`：
```
>	gpuApi.dll!GpuApi::Present(const GpuApi::SwapChainRef & swapChain, bool useVsync, unsigned int vsyncThreshold) 行 310	C++
 	render.dll!RenderViewport::Present() 行 586	C++
 	render.dll!CRenderNode_Present::Execute(const SRenderNodeImplContext & rctx, job::Builder * builder) 行 369	C++
 	render.dll!CRenderNodeBase::Process(const SRenderNodeImplContext & rctx, job::Builder * builder) 行 172	C++
 	render.dll!helper::RunRenderJob(const helper::RunRenderJobParameter & param, const job::RunContext & runContext) 行 4279	C++
 	render.dll!helper::DispatchRenderJobs::__l6::<lambda>(const job::RunContext & context) 行 4427	C++
 	render.dll!job::prv::JobShim<void <lambda>(const job::RunContext &),rend::PoolRenderGraph>::RunJob(void * jobData, const job::RunContext & runContext) 行 33	C++
 	redJobs2.dll!job::prv::helper::RunJobFunctionWithInstrumentation(const job::prv::JobQueueEntry & entry, const job::RunContext & runContext) 行 640	C++
 	redJobs2.dll!job::prv::Dispatcher::DoRunJobQueueEntry(const job::prv::JobQueueEntry & entry, const unsigned int dispatcherThreadIndex, job::Priority priority, job::prv::JobScopeMemoryAllocator & jobScopeAllocator, red::CircularBuffer<std::pair<job::prv::JobQueueEntry,enum job::Priority>> * optLocalQueue) 行 700	C++
 	redJobs2.dll!job::prv::DispatcherThread::DoWorkLoop() 行 93	C++
 	redJobs2.dll!job::prv::DispatcherThread::ThreadFunc() 行 58	C++
 	redSystem.dll!red::WinAPI::ThreadEntryFunc(void * userData) 行 228	C++

```