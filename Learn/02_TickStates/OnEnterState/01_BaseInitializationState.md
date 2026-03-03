- **底层模块初始化**，包括内存池、网络模块（条件编译下）、日志系统，并在非最终版中打印命令行参数，同时对 `-GOGPassword` 做脱敏处理，避免敏感信息泄露。
- 初始化 **IO 系统** 和 **HTTP 系统**，根据命令行决定是否开启 IO profiling，同时设置 CPU 浮点模式，以提升运行时数值处理性能和稳定性。
- 解析启动参数生成 `appSettings`，处理诸如**等待调试器附加**、**启动时主动断点中断**之类的调试行为；然后初始化 profiler、主线程 profiler、job 线程系统，为后续多线程任务调度做准备。
- 调用 `gameApp.Initialize(appSettings)` 完成具体游戏实例初始化，设置内存预算，并继续初始化基础引擎系统。
- 如果中间某些关键模块初始化失败，则会记录 fatal 日志并返回失败。

```c++
	bool GameAppBaseInitializationState::OnEnterState( GameAppInstance& gameApp )
	{
		const red::CommandLine& commandLine = red::CommandLine::Get();

		// Initialize memory pools for modules without initialization
		InGameConfig::InitializeMemoryPools();
		img::memory::InitializeMemoryPools();
		compression::InitializeMemoryPools();

#if defined( RED_NETWORK_ENABLED )
		red::Network::InitializeMemoryPools();
		red::Network::Base::Initialize();
#endif
		//-----------------------

		const auto& processPrettyName = GetLogPrettyName( commandLine );
		app::logging::Initialize( commandLine, processPrettyName, gameApp.GetName(), GetLogDescription( commandLine ) );
		app::logging::SetFrameNumberRetriever( GetCurrentFrameNumber );

		//-----------------------

#if !defined( RED_CONFIGURATION_FINAL )
		{
			red::String commandLineToPrint = String_CreateExternal_OnStack( 4096 );
			commandLineToPrint = red::CommandLine::Internal_GetRawCommandLine();
			Uint32 GOGPasswordStartIndex = -1;
			const char GOGPasswordArgStart[] = "-GOGPassword \"";
			if ( commandLineToPrint.IndexOf( GOGPasswordArgStart, GOGPasswordStartIndex ) )
			{
				GOGPasswordStartIndex += sizeof( GOGPasswordArgStart ) - 1;
				Uint32 GOGPasswordEndIndex = -1;
				if ( commandLineToPrint.IndexOf( '\"', GOGPasswordEndIndex, GOGPasswordStartIndex ) )
				{
					red::Memset( &commandLineToPrint[ GOGPasswordStartIndex ], '*', GOGPasswordEndIndex - GOGPasswordStartIndex );
				}
			}
			RED_LOG_INFO( "cmd: %s", commandLineToPrint.AsChar() );
		}
#endif

		//-----------------------

		io::InitSetup ioSetup;
		if ( commandLine.HasOption( "profileio" ) )
		{
			ioSetup.enableProfiler = true;
		}

		if ( !io::Initialize( ioSetup ) )
		{
			RED_FATAL( "Failed to initialize redIO" );
			return true;
		}

		if ( ioSetup.enableProfiler )
		{
			io::GAsyncIO.ProfileStartTrace();
		}

		//-----------------------

		http::InitParams initParams;
		initParams.maxWorkers = 1;
		if( !http::Initialize( initParams ) )
		{
			RED_FATAL( "Failed to initialize redHTTP" );
			return false;
		}

		//-----------------------

		_MM_SET_FLUSH_ZERO_MODE(_MM_FLUSH_ZERO_ON);
		_MM_SET_DENORMALS_ZERO_MODE(_MM_DENORMALS_ZERO_ON);

		//-----------------------

		app::InitializationSettings appSettings;
		appSettings.ParseFromCommandline( commandLine );

		if ( appSettings.m_waitForDebugger )
		{
			while ( !dbgutils::IsDebuggerAttached() )
			{
				continue;
			}
		}

		// Don't care if debugger attached - maybe on purpose so get Windows dialog so you can attach a debugger this way
		// But of course check after the above prompt/wait for debugger setting.
		if ( appSettings.m_debugBreak )
		{
			RED_BREAKPOINT();
		}

		//-----------------------

#ifdef USE_PROFILER
		red::profiler::InitInGameProfiler();
#endif
 		PROFILER_InitThread( "Main", 16384 );

		job::InitParam jobInitParam;
		if( appSettings.m_jobThreadCount != -1 )
		{
			jobInitParam.maxThreads = appSettings.m_jobThreadCount;
		}
		job::Initialize( jobInitParam );

		//-----------------------

		if ( !gameApp.Initialize( appSettings ) )
		{
			return false;
		}

		red::SetMemoryBudgets();

		InitializeBaseEngineSystems( gameApp, commandLine );
		return true;
	}
```