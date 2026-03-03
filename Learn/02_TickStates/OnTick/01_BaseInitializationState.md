```c++
bool GameAppBaseInitializationState::OnTick( GameAppInstance& gameApp )
{
	if ( !GEngine )
	{
		// pre initialization failed
		gameApp.RequestNextStateTransition( GameAppState_Shutdown );
		return true;
	}

	Bool result = false;

	const EBaseEngineState state = GEngine->GetState();
	switch ( state )
	{
	case BES_Uninitialized:
		RED_LOG( "GameAppBaseInitializationState: Failed to initalize engine %hs", GEngine->GetClass()->GetName().AsChar() );
		gameApp.RequestNextStateTransition( GameAppState_Shutdown );
		result = true;
		break;

	case BES_InitializingRootSystems:
		red::YieldCurrentThread();
		break;

	case BES_InitializingRegularSystems:
		// finished initialization of the root systems, so we can start ticking simplified frame (renderer, audio, input and few others)
		gameApp.RequestNextStateTransition( GameAppState_Initialization );
		result = true;
		break;

	case BES_Initialized:
		gameApp.RequestNextStateTransition( GameAppState_GameRunning );
		break;

	default:
		RED_FATAL_ASSERT( 0 , "GameAppBaseInitializationState: Invalid engine state '%u'", state );
		break;
	}

	return result;
}
```