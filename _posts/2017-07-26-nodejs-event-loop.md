- libuv
- There is only one thread
- Event loop runs in that one thread
- Async work done by async I/O interfaces provided by most operating systems. Only when that is not available does node fallback to a thread pool (with 4 threads).
- Event Loop Phases:
  1. Timers
  1. IO Callbacks
  1. IO Polling
  1. Set Immediate
  1. Close Events
