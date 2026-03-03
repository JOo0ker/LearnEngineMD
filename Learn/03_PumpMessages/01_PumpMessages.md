使用Windows API获取消息
```c++
Bool PlatformPumpMessages()
{
	MSG msg;
	while ( ::PeekMessageW( &msg, nullptr, 0, 0, PM_REMOVE ) )
	{
		if ( msg.message == WM_QUIT )
		{
			return false;
		}

		::TranslateMessage(&msg);
		::DispatchMessageW(&msg);
	}

	return true;
}
```
---
```c++
bool AppInstance::PumpMessages()
{
	// Platform specific code
	return platform::PlatformPumpMessages();
}
```