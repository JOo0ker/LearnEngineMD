
```c++
bool GameAppInitializationState::OnEnterState( GameAppInstance& gameApp )
{
	RED_ASSERT( GEngine, "GameAppInitializationState: Engine must exist at this state" );

	if ( red::MultiplayerSetup::IsServer() )
	{
		// there is no video player on the server and probably we do not want to display any splash screen
		return true;
	}

	if ( auto viewportWrapper = GetInkSystem().GetGameViewportWrapper() )
	{
		viewportWrapper->QueueViewportEvent( CreateHandle< ink::StateChangeRequestEvent >( RED_NAME_CONSTEXPR( "inkInitEngineState" ) ) );
	}

	return true;
}
```