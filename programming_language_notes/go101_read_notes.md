# Go101 Read Notes
## initialization order
- imported packaged
- global variables 
- init() functions

## Goroutine
- maintained and scheduled by go runtime
- light-weighted, 2KB stack each
- each program always starts from the main goroutine, and when such goroutine exits, the program exits regardless of any other goroutines

## MPG model
- `M` stands for OS-threads. It has nothing to do with level of parallelism. You may consider it as a worker.

- `P` stands for logical Processors. Its value is determined by `GOMAXPROCS`. Each logical processor owns a run queue of goroutines, which is generally scheduled FIFO. A goroutine has a maximum 10 ms duration before getting preempted.

- `G` stands for goroutines.

### work stealing
You may think in this way: there is an advanced thread pool enabling task stealing. Such pool holds `GOMAXPROCS` threads, meaning that at most this number of threads can run in parallel. Each thread has its own task queue. A thread sequentially takes a task from its queue and executes it. When the queue is empty, the thread may randomly select a victim thread, from which it steals half the tasks.

### advantages over os-level thread
1. light-weighted: the default stack size of a goroutine is 2KB, but for an os thread, it's 8MB (linux, macos) or 1MB(windows).
2. lower creation cost
3. cheaper context switching cost: it is done by go runtime, do not need to switch between kernel state and user state.
4. M:N scheduling: a single P manages multiple goroutines