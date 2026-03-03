
```c++
Bool CBaseEngine::ProcessMainLoopFrame( const Float timeDelta )
{
	red::Timer timer;
	const Uint64 beginFrameTick = timer.GetTicks();

	Bool result = true;

#if defined( RED_ENABLE_EVENT_DEBUG )
	red::eventDebug::OnBeginFrame();
#endif

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

	if (GResourceLoader)
	{
		GResourceLoader->Update();
	}

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
			GRender->Tick( timeDelta, tickInfo.m_globalTimeDilation, m_lastFrameCPUTime );
		}
	}

	red::UniquePtr<input::InputContextImpl> inputContext;

	// Update inputs
	{
		PC_SCOPE( ProcessInput );
		if ( m_inputSystem )
		{
			inputContext = input::GatherInput(m_inputSystem.Get(), scaledTimeDelta); // TODO: should we cheat the input? pass scaled timeDelta?

			ProcessInputInternal( *inputContext );
		}
	}

	if ( m_inputSystem )
	{
		if( m_inkSystem )
		{
#ifdef INK_ENABLE_PROFILING
			m_inkSystem->SetCurrentFrameNumber( GetCurrentEngineTick() );
#endif
			m_inkSystem->BeginFrame();
			m_inkSystem->Tick( tickInfo, *inputContext.Get() );
		}

		if ( m_soundSystem && inputContext->GetBufferedInputData().Size() > 0 )
		{
			m_soundSystem->SetUsingKeyboardAndMouse(
				inputContext->GetDeviceType( inputContext->GetBufferedInputData().Back().m_deviceID ) == input::KBD_MOUSE );
		}

		// Process simulation frame
		{
			result = ProcessFrame( scaledTimeDelta, tickInfo, *inputContext.Get() );
		}
	}

#ifndef NO_RUNTIME_MATERIAL_COMPILATION
	if ( m_asyncMaterialCompiler )
	{
		m_asyncMaterialCompiler->Tick( scaledTimeDelta, tickInfo );
	}
#endif // NO_RUNTIME_MATERIAL_COMPILATION

	// Process resource monitoring events
	{
		PC_SCOPE(ResourceMonitor);
		res::MonitorRouter::GetInstance().DispatchChanges();
	}

	{
		PC_SCOPE( InGameConfigSystem );
		InGameConfig::System::GetInstance().Update();
	}

	// Update last frame CPU time
	const Uint64 endFrameTick = timer.GetTicks();
	m_lastFrameCPUTime = static_cast< Float >( ( endFrameTick - beginFrameTick ) / timer.GetFrequency() * 1000.f );

	// Update current engine tick
	++m_currentEngineTick;

	return result;
}

```