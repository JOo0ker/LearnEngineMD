带有时间管理、性能统计、内存帧回收和单帧逻辑调度的引擎主循环单步执行函数

```c++
Bool CBaseEngine::LoopSingleTick( Bool ( CBaseEngine::* singleTickFun )( const float ) )
{
	Bool result = false;

	{
		PC_SCOPE( Frame );

#if defined( RED_LOGGING_ENABLED )
		UpdateLoggingDebugFilters();
#endif

#ifdef USE_RED_INGAME_PROFILER
	if ( !IsHeadless() )
	{
		m_inGameProfilerFrontEnd->Update();
	}
#endif

#ifdef CONTINUOUS_SCREENSHOT_HACK
		red::Clock::GetInstance().GetTimer().NextFrameGameTimeHack();
#endif

		// Calculate time delta
		Double curTime = red::Clock::GetInstance().GetTimer().GetSeconds();
		m_lastTimeDeltaUnclamped = (Float)( curTime - m_oldTickTime );
		m_oldTickTime = curTime;

#ifdef RED_CONFIGURATION_FINAL
		float timeDelta = Min( m_lastTimeDeltaUnclamped, 0.066f );
#else
		float timeDelta = Min( m_lastTimeDeltaUnclamped, 0.1f ); // so it is "playable" with low framerate - although issues related to physics may appear
#endif

		// Advance internal timers
		m_rawEngineTime += timeDelta;
		m_lastTimeDelta = timeDelta;

		Float currentTickRate = 1.f / timeDelta;

		if (m_minTickRateOver1Sec > currentTickRate || m_minTickRateOver1Sec == 0.f)
		{
			m_minTickRateOver1Sec = currentTickRate;
		}

		// Tick rate calculation
		if ( curTime > (m_prevTickRateTime + 1.0) )
		{
			m_lastTickRate = m_numTicks / (Float)( curTime - m_prevTickRateTime );
			m_minTickRateToShow = m_minTickRateOver1Sec;
			m_minTickRateOver1Sec = 0.f;
			m_prevTickRateTime = curTime;
			m_numTicks = 0;
		}

		// Tick the engine
		if ( timeDelta > 0.0f )
		{
			result = ( this->*singleTickFun )( timeDelta );
		}

		// Count ticks
		m_numTicks += 1;
	}

	// Notify redMemory about new frame and reset frame-based allocators
	red::memory::ProcessNextFrame();

	// right out of scope, so the main frame scope is closed when the frame ticks in the profiler
	PROFILER_NextFrame( red::ProfilerFrameType::PFT_ENGINE );

	return result;
}
```