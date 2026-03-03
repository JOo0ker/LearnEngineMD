
```c++
	bool GameAppShutdownState::OnTick( GameAppInstance& gameApp )
	{
		Int32 returnCode = ShutdownEngine();
		gameApp.SetReturnValue( returnCode );

		gameApp.Shutdown();

		http::Shutdown();
		app::logging::Shutdown();
		io::Shutdown();

		//job::Shutdown();

#if defined( RED_NETWORK_ENABLED )
		red::Network::Base::Shutdown();
#endif
		return true;
	}
```