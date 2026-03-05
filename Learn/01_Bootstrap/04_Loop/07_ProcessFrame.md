
```c++
Bool CGameEngine::ProcessFrame( const Float scaledEngineTimeDelta, const tick::Info& tickInfo, input::IInputContext& input )
{
	PC_SCOPE( CGameEngine_ProcessFrame );

	Float frameDeltaTime = scaledEngineTimeDelta;
	Bool result = true;

	result = TBaseClass::ProcessFrame( frameDeltaTime, tickInfo, input );

#ifdef RED_USE_IMGUI
	m_viewport->ImGuiNewFrame( tickInfo.m_realTimeDelta );
#endif

#if defined( RED_SAVE_SERVER_ENABLED )
	red::SaveServer::GetInstance().Update( frameDeltaTime, input );
#endif

	// start rendering
	CRenderFrameInfo frameInfo;
	CRenderCameraFrameInfo cameraInfo;
	m_viewport->BeginFrame( frameInfo, cameraInfo );
	{
		const Float engineTime = GetRawEngineTime();
#ifdef RED_MEMORY_ENABLE_REPORT_EXTRAS
		red::memory::Debug_SetPlayTIme( engineTime );
#endif 
		frameInfo.SetEngineTime( engineTime );
		frameInfo.SetCleanEngineTime( engineTime );
	}

	// handle debug update of the master viewport
	m_viewport->ProcessDebugInput( input );
	m_viewport->UpdateDebug( tickInfo.m_realTimeDelta ); // don't cheat the UI with scaled timeDelta

	m_services->Update();
	UpdateGOGRewardsSystem( tickInfo.m_realTimeDelta );

	// setup game frame
	gsm::Frame frame =
	{
		frameDeltaTime,
		tickInfo,
		input,
		&frameInfo,
		&cameraInfo
	};

	// get the frame messages from input
	m_inputSystem->GetMessages(frame.m_messages);

	// push all messages from services into the frame and clear services message queue
	m_services->GetMessages(frame.m_messages);

	// Force pause for limited resource mode
	const EBaseEngineResourceMode resourceMode = GetResourceMode();
	if( resourceMode == BERM_FullWithSystemNoInput || resourceMode == BERM_Constrained )
	{
		frame.m_messages.PushBack( { RED_NAME_CONSTEXPR( "PauseBackground" ), {} } );
	}

	// process the free camera
	ProcessFreeCamera( frame );

	// process the video player
	m_videoPlayer->Tick();

	m_fadeOutSupervisor->Tick( tickInfo.m_realTimeDelta );
#ifdef RED_USE_IMGUI
	m_fadeOutSupervisor->RenderDebug( cameraInfo.GetDebugDrawer() );
#endif

	// Tick the game state machine
	if ( !( result = m_gameFramework->ProcessFrame( frame ) ) )
	{
		// normal exit
		RED_LOG_INFO( "CGameEngine: ProcessFrame failed. Requesting for a game exit" );
		RequestExit( 0 );
	}

#ifdef RED_USE_IMGUI
	m_viewport->ImGuizmoUpdate( cameraInfo.GetRenderCamera() );
#endif

	GRender->GetCommands()->FlushPreviousFrameCommandsProcessing();

	red::DynArray<TRenderPtr<IRenderScene>> scenesForUpdate{ red::PoolFrame() };
	GatherScenesForUpdate( &scenesForUpdate, frameInfo );
	GRender->GetCommands()->FrameTick( scenesForUpdate );

	//we have no scene so we need to ask UI to render by itself
	if (scenesForUpdate.Size() == 0)
	{
		GetInkSystem().SimpleRender( frameInfo );
	}

#ifdef RED_USE_IMGUI
	m_viewport->DrawImGui( frameInfo.m_systemOnRenderDebug, cameraInfo.GetDebugDrawer() );
	m_viewport->ImGuiEndFrame( frameDeltaTime );
#endif

#if !defined( RED_CONFIGURATION_FINAL ) || defined( USE_PROFILER )
	FT_CollectStats( frameInfo, cameraInfo );
#endif

#ifdef RED_MEMORY_ENABLE_REPORT_EXTRAS
	red::memory::TryDoAutoMemReport(frameDeltaTime);
#endif

	// Render collected frame into the master viewport
	// #tbd: not ideal above that we wait for the tick to finish completely before submitting the frame
	m_viewport->SubmitFrame( frameInfo, cameraInfo );

	return result;
}

```