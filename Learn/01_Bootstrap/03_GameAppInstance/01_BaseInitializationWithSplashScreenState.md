# 基础初始化启动画面状态

```c++
	class GameAppBaseInitializationWithSplashScreenState final : public red::GameAppBaseInitializationState
	{
	public:
		virtual bool OnEnterState( red::GameAppInstance& gameApp ) override
		{
			red::InitializeSplashScreen();
			return red::GameAppBaseInitializationState::OnEnterState( gameApp );
		}
	};
```

^6f7786
