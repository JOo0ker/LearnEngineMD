# 关闭状态

```c++
	class GameAppShutdownState final : public IGameAppState
	{
	public:
		GameAppShutdownState();
		~GameAppShutdownState();

		virtual const char* GetName() const override;
		virtual Int32 GetId() const override;

		virtual bool OnTick( GameAppInstance& gameApp ) override;
	};
```