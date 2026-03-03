# 根据当前状态抉择处理逻辑

- State
> [[Learn/02_TickStates/OnEnterState/01_BaseInitializationState|01_BaseInitializationState]][[01_BaseInitializationWithSplashScreenState#^6f7786]]
> [[Learn/02_TickStates/OnEnterState/02_InitializationState|02_InitializationState]]

- Tick
> [[Learn/02_TickStates/OnTick/01_BaseInitializationState|01_BaseInitializationState]]
> [[Learn/02_TickStates/OnTick/02_InitializationState|02_InitializationState]][[02_InitializationWithSplashScreenState#^3bb792]]
> [[03_Running]]
> [[04_Shutdown]]

- Exit
> [[01_InitializationState]]
---
```c++
void GameAppInstance::TickStates()
{
	if ( m_currentState )
	{
		switch ( m_currentContext )
		{
		case StateContext::StateContext_EnterState:
			if ( m_currentState->OnEnterState( *this ) )
			{
				m_currentContext = StateContext::StateContext_Tick;
			}
			else
			{
				m_currentContext = StateContext::StateContext_ExitState;
			}
			break;

		case StateContext::StateContext_Tick:
			if ( m_currentState->OnTick( *this ) )
			{
				m_currentContext = StateContext::StateContext_ExitState;
			}
			break;

		case StateContext::StateContext_ExitState:
			m_currentState->OnExitState( *this );
			m_currentContext = StateContext::StateContext_NextState;
			break;

		case StateContext::StateContext_NextState:
			m_currentState = m_nextState;
			m_currentContext = StateContext::StateContext_EnterState;
			m_nextState = nullptr;
			break;
		}
	}
	else if ( m_nextState )
	{
		m_currentState = m_nextState;
		m_currentContext = StateContext::StateContext_EnterState;
		m_nextState = nullptr;
	}
}
```