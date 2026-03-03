# 主程序循环
1. 首先处理消息
2. 运行帧逻辑并更新当前状态
3. 根据状态判断是否结束循环
[[01_TickStates]]
```c++
Int32 GameAppInstance::Run()
{
	do
	{
		if ( !PumpMessages() )
		{
			RED_LOG_ERROR( "[GameApplication] PumpMessages failed. Requesting for game exit" );
			GEngine->RequestExit( -66 );
		}

		TickStates();
	} while ( m_currentState );

	return m_returnValue;
}
```