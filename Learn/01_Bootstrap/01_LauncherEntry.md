# 主程序入口，程序启动时最先进入此函数
- launcher/Windows/mainWindows.cpp
```c++
int WINAPI wWinMain( HINSTANCE hInstance, HINSTANCE hPrevInstance, wchar_t* commandLine, int nCmdShow )
{
	RED_CHECK_FOR_MULTIPLE_INSTANCES();
	
	SetProcessDPIAware();

	red::CommandLine::Init( commandLine );
	const int programReturnValue = mainRed();

	RED_CHECK_FOR_MULTIPLE_INSTANCES_CLEANUP();

	// ctremblay: game is correctly shutdown, now we can safely ignore other issue and quickly exit.
	std::quick_exit(programReturnValue);
	return programReturnValue;
}
```
# 程序准入口
```c++
int mainRed()
{
	int ret = 1;
	ret = mainRedGuarded();
	return ret;
}
```
# 真实入口，包含各类初始化和状态注册
- launcher/Common/mainRed.cpp
 >[[01_LowLevelSystems]]
> [[02_ModuleDependencies]]
> [[00_AppInstance]] 
> [[01_BaseInitializationWithSplashScreenState]]
> [[02_InitializationWithSplashScreenState]]
> [[03_RunningState]]
> [[04_ShutdownState]]
> [[99_Run]]

```c++
	int mainRedGuarded()
	{
		red::InitializeLowLevelSystems();
		
		InitModuleDependencies();

		red::GameAppInstance game{ "launcher" };

		GameAppBaseInitializationWithSplashScreenState baseInitStaste;
		game.RegisterState( &baseInitState );

		GameAppInitializationWithSplashScreenState initState;
		game.RegisterState( &initState );

		red::GameAppRunningState gameRunningState;
		game.RegisterState( &gameRunningState );

		red::GameAppShutdownState shutdownState;
		game.RegisterState( &shutdownState );

		game.RequestNextStateTransition( red::GameAppState_BaseInitialization );

		return game.Run();
	}
}
```
