
[[03_LoopSingleTick]]

```c++
Bool CBaseEngine::MainLoopSingleTick()
{
	return LoopSingleTick( &CBaseEngine::ProcessMainLoopFrame );
}
```