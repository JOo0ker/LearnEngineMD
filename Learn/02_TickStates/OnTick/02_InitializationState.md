```c++
bool GameAppInitializationState::OnTick( GameAppInstance& gameApp )
{
	Bool result = false;

	const EBaseEngineState state = GEngine->GetState();
	switch ( state )
	{
	case BES_Uninitialized:
		RED_LOG( "GameAppInitializationState: Failed to initalize engine %hs", GEngine->GetClass()->GetName().AsChar() );
		gameApp.RequestNextStateTransition( red::GameAppState_Shutdown );
		result = true;
		break;

	case BES_InitializingRegularSystems:
		// finished initialization of the root systems, so we can tick simplified frame (renderer, audio and input and few others)
		GEngine->BaseLoopSingleTick();
		break;

	case BES_Initialized:
		RED_LOG( "GameAppInitializationState: Initalized engine %hs", GEngine->GetClass()->GetName().AsChar() );
		gameApp.RequestNextStateTransition( red::GameAppState_GameRunning );
		result = true;
		break;

	case BES_Suspended:
		RED_LOG( "GameAppInitializationState: Initialization suspended" );
		break;

	default:
		RED_FATAL_ASSERT( 0 , "GameAppInitializationState: Invalid engine state '%u'", state );
		gameApp.RequestNextStateTransition( red::GameAppState_Shutdown );
		result = true;
		break;
	}

	return result;
}
```