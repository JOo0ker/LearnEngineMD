初始化低层系统，包括以下内容：
- 初始化错误处理
- 初始化Root池
- 初始化Memory池
- 注册主线程
- 初始化多人模式
可选初始化以下内容：
- 终端默认分配
- 初始化RTTIService
- 初始化序列化调试
```c++
	void InitializeLowLevelSystems()
	{
		red::InitErrorHandler( red::eErrorHandlerFlags_Default, &VisitScriptCallstacks );
		
		//-----------------------

		red::memory::InitializeRootPools();
		red::InitializeCoreMemoryPools();

		//-----------------------

		const red::CommandLine& commandLine = red::CommandLine::Get();

		if ( commandLine.HasOption( "breakOnPoolDefault" ) )
		{
			red::memory::BreakOnPoolDefaultAllocation();
		}

		RED_FATAL_ASSERT( ::SIsMainThread(), "InitializeLowLevelSystems must be called from main thread" );
		red::memory::RegisterCurrentThread( "Main Thread" );

		red::MultiplayerSetup::Initialize();
		if( red::MultiplayerSetup::IsMultiplayer() )
		{
			// need to initialize it before modules initialization
			rep::RTTIService::Initialize();
		}

#ifdef RED_ENABLE_SERIALIZABLE_DEBUG
		serializableDebug::Initialize();
#endif
	}
```