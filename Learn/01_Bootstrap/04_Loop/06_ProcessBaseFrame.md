
```c++
Bool CGameEngine::ProcessBaseFrame( const Float scaledEngineTimeDelta, const tick::Info& tickInfo, input::IInputContext& input )
{
	Bool result = true;

	Float frameDeltaTime = scaledEngineTimeDelta;

#ifndef RED_CONFIGURATION_FINAL
	if (IsGameMode())
	{
		res::SetDebugWaitUntilLoadedRenderCallback([this](const res::ResourcePath& waitingForResource, Float waitingSeconds) {	
			rend::StalledFrameParams params;
			params.waitingForResource = waitingForResource;
			params.waitingSeconds = waitingSeconds;
			SubmitStalledFrameForDebug(params);
		});
	}
#endif

	result = TBaseClass::ProcessFrame( frameDeltaTime, tickInfo, input );

#ifdef RED_USE_IMGUI
	m_viewport->ImGuiNewFrame( tickInfo.m_realTimeDelta );
#endif

	// start rendering
	CRenderFrameInfo frameInfo;
	CRenderCameraFrameInfo cameraInfo;
	m_viewport->BeginFrame( frameInfo, cameraInfo );
	{
		const Float engineTime = GetRawEngineTime();
		frameInfo.SetEngineTime( engineTime );
		frameInfo.SetCleanEngineTime( engineTime );
	}

#ifdef RED_USE_IMGUI
	m_viewport->ImGuizmoUpdate( cameraInfo.GetRenderCamera() );
#endif

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
	m_inputSystem->GetMessages( frame.m_messages );

	// push all messages from services into the frame and clear services message queue
	m_services->GetMessages( frame.m_messages );


	// process the free camera
	ProcessFreeCamera( frame );

	// process the video player
	m_videoPlayer->Tick();

	GRender->GetCommands()->FlushPreviousFrameCommandsProcessing();

	red::DynArray<TRenderPtr<IRenderScene>> scenesForUpdate{ red::PoolFrame() };
	GatherScenesForUpdate( &scenesForUpdate, frameInfo );
	GRender->GetCommands()->FrameTick( scenesForUpdate );

	GetInkSystem().SimpleRender( frameInfo );

#ifdef RED_USE_IMGUI
	m_viewport->DrawImGui( frameInfo.m_systemOnRenderDebug, cameraInfo.GetDebugDrawer() );
	m_viewport->ImGuiEndFrame( frameDeltaTime );
#endif

	// Render collected frame into the master viewport
	// #tbd: not ideal above that we wait for the tick to finish completely before submitting the frame
	m_viewport->SubmitFrame( frameInfo, cameraInfo );

	return result;
}

```