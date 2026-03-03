# 初始化启动画面状态

```c++
class GameAppInitializationWithSplashScreenState final : public red::GameAppInitializationState
{
public:
	virtual bool OnEnterState( red::GameAppInstance& gameApp ) override
	{
		if ( !gameApp.m_engineInitParams.m_isBackendEngine )
		{
			return red::GameAppInitializationState::OnEnterState( gameApp );
		}

		return true;
	}

	virtual bool OnExitState( red::GameAppInstance& gameApp ) override
	{
		red::UninitializeSplashScreen();

		if ( !gameApp.m_engineInitParams.m_isBackendEngine )
		{
			return red::GameAppInitializationState::OnExitState( gameApp );
		}

		return true;
	}
};
```

^3bb792
