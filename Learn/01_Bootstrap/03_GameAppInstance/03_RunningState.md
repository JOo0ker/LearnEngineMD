# 运行状态

```c++
	class GameAppRunningState final : public IGameAppState
	{
	public:
		GameAppRunningState();
		virtual ~GameAppRunningState() override;

		virtual const char* GetName() const override;
		virtual Int32 GetId() const override;

		virtual bool OnEnterState( GameAppInstance& gameApp ) override;
		virtual bool OnTick( GameAppInstance& gameApp ) override;
	};
```