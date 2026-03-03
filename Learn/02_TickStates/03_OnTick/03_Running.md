```c++
bool GameAppRunningState::OnTick( GameAppInstance& gameApp )
{
	if ( !GEngine->MainLoopSingleTick() )
	{
		if ( !GEngine->RequestedExit() )
		{
			GEngine->RequestExit( -42 );
		}

		gameApp.RequestNextStateTransition( GameAppState_Shutdown );
		return true;
	}

	return false;
}
```