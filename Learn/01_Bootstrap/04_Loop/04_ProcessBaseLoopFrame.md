
```c++
Bool CBaseEngine::ProcessBaseLoopFrame( const Float timeDelta )
{
	red::Timer timer;
	const Uint64 beginFrameTick = timer.GetTicks();

	Bool result = true;

#if defined( USE_PROFILER )
	CScriptingSystem::GetInstance().ResetLastScriptFrameDuration();
#endif

	if ( Config::cvDebugDumpMultiplayer.Get() )
	{
		static Uint32 frameIndex = 0;
		RED_LOG( "=== Next engine frame %u ===", frameIndex++ );
	}

	telemetry::Update();

	if ( !IsHeadless() )
	{
		// If renderer is not ready eg. has lost device, do not tick game
		PC_SCOPE( PrepareRenderer );
		if ( !GRender->PrepareRenderer() )
		{
			PC_SCOPE( RenderNotPrepared );
			red::SleepOnCurrentThread( 100 );
			return result;
		}
	}

#ifdef RED_NETWORK_ENABLED
	// update debug server
	DBGSRV_CALL( Update() );
#endif

	// Update debug page service
#ifdef RED_NETWORK_ENABLED
	{
		PC_SCOPE( DebugPageServer );
		CDebugPageServer::GetInstance().ProcessServiceCommands( timeDelta );
	}
#endif

	// Advance the screenshot system
	{
		PC_SCOPE( SScreenshotSystem_Tick );
		GetScreenshotSystem().Tick();
	}

	// Process time dilation
	const Uint32 flags = GetFadeOutSupervisor() ? GetFadeOutSupervisor()->GetTickFlags() : 0;
	const tick::Info tickInfo( timeDelta, m_timeDilation->ComputeGlobalTimeDilation(), *m_timeDilation, flags );
	const Float scaledTimeDelta = timeDelta * tickInfo.m_globalTimeDilation;

	{
		UpdateStreamingFallback( tickInfo );
	}

	// Tick renderer
	{
		PC_SCOPE( RenderTick );
		if ( GRender )
		{
			GRender->Tick( scaledTimeDelta, tickInfo.m_globalTimeDilation, m_lastFrameCPUTime );
		}
	}

	red::UniquePtr<input::InputContextImpl> inputContext;

	// Update inputs
	{
		PC_SCOPE( ProcessInput );
		inputContext = input::GatherInput(m_inputSystem.Get(), scaledTimeDelta); // TODO: should we cheat the input? pass scaled timeDelta?

		ProcessInputInternal( *inputContext );
	}

	// Process UI framework
	if ( m_inkSystem )
	{
		m_inkSystem->SimpleTick( tickInfo, *inputContext.Get() );
	}

	// Process simulation frame
	{
		PC_SCOPE( ProcessBaseFrame );
		result = ProcessBaseFrame( scaledTimeDelta, tickInfo, *inputContext.Get() );
	}

	// Process resource monitoring events
	{
		PC_SCOPE(ResourceMonitor);
		res::MonitorRouter::GetInstance().DispatchChanges();
	}

	// Update last frame CPU time
	const Uint64 endFrameTick = timer.GetTicks();
	m_lastFrameCPUTime = static_cast< Float >( ( endFrameTick - beginFrameTick ) / timer.GetFrequency() * 1000.f );

	// Update current engine tick
	++m_currentEngineTick;

	return result;
}

```
