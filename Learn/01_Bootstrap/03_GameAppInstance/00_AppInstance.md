# 应用实例
使用`GameAppState`代表各类状态

```c++
	enum GameAppState
	{
		GameAppState_BaseInitialization = 0,
		GameAppState_Initialization,
		GameAppState_GameRunning,
		GameAppState_Shutdown
	};
	
	class GameAppInstance : public app::AppInstance
{
public:
	GameAppInstance( const char* processName );
	~GameAppInstance();

	const char* GetName() const;

	void RequestNextStateTransition( Int32 stateId );
	bool RegisterState( IGameAppState* state );
	Int32 Run();

	void SetReturnValue( Int32 value );

	EngineInitParams m_engineInitParams;

private:
	virtual bool Initialize_InGameConfig( const app::InitializationSettings& settings ) override;
	virtual void Shutdown_InGameConfig() override;
	virtual void Initialize_Censorship( const app::InitializationSettings& settings ) override;
	virtual void Shutdown_Censorship() override;
	virtual void Initialize_Telemetry( const app::InitializationSettings& settings ) override;
	virtual void Shutdown_Telemetry() override;

	void TickStates();

	red::Map< Int32, IGameAppState* > m_states;

	enum class StateContext
	{
		StateContext_EnterState,	// Keep calling enter state on the current state
		StateContext_Tick,			// Keep calling tick on the current state
		StateContext_ExitState,		// Keep calling exit state on the current state
		StateContext_NextState		// Finished with this state, go to the next one
	};

	IGameAppState* m_currentState;
	IGameAppState* m_nextState;
	StateContext m_currentContext;

	const char* m_processName;
	Int32 m_returnValue;
};
```