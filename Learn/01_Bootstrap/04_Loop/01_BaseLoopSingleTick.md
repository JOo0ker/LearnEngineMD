
[[03_LoopSingleTick]]
[[04_ProcessBaseLoopFrame]]

```c++
Bool CBaseEngine::BaseLoopSingleTick()
{
	return LoopSingleTick( &CBaseEngine::ProcessBaseLoopFrame );
}
```