# C++ Concurrency in Action Read Notes
## Arguments of std::thread
### Single Argument
std::thread's constructor takes in any callable object, namely:
- Function pointer
- Lambda
- Functor Object(a class which implements operator())
- std::bind expression

For a function object, it must overload the call operator and mark it as public.
```c++
class FooCallable {
public:
    // must be public
    void operator()() const {
        cout << "foo calls" << endl;
    }
};

int main() {
    FooCallable fc;
    std::thread thd(fc);
    thd.join();
    return 0;
}
```
The `most vexing parse` refers to the fact that when you directly create a temporary function object as the sole argument to a constructor using parentheses, the compiler may interpret it as a function declaration instead of an object construction.

For instance, if you write like:
```c++
int main() {
    std::thread thd(FooCallable()); // this is a function declaration
    thd.join(); // Compiler: "Cannot resolve symbol 'join'"
    return 0;
}
```
The first statement is construed as "declares a function named thd that returns a std::thread and takes a function pointer `FooCallable (*)()` as parameter". The solution to this problem is either using named object or utilize these two forms:
```c++
std::thread thd((FooCallable())); // enclosed it in a pair of brackets
std::thread thd{FooCallable()}; // use curly brace
```
### Multiple Argument 
Another constructor of `std::thead` takes in multiple arguments, The first is a callable object, and the rest are arguments to be passed to that callable.

#### Two-Phased Assignment
>  _" it’s important to bear in mind that by default, the arguments are **COPIED** into internal storage, where they can be accessed by the newly created thread of execution, and then passed to the callable object or function as **rvalues** as if they were **temporaries**. This is done even if the corresponding parameter in the function is **expecting a reference**. "_

What it means is that, `std::thread` does not consult the callable's signature when you construct it. The construction and invocation happen in two distinct phases:  

- When we create an instance of std::thread by calling its constructor, the constructor blindly value-copies all arguments into its own buffer, because, in fact, the library use `std::decay_t` to store the arguments.

- When the thread triggers the callable, parameters in buffer are moved as `rvalue` onto the callable parameters. As a result, the callable sees arguments as temporaries — even if it expects references.

For instance, the following code won't compile:

```c++
class FooCallable {
public:
    void operator()(int& i) const {
        int tmp = i;
        i = 200;
        printf("foo changes i from %d into %d\n", tmp, i);
    }
};

int main() {
    int input = 100;
    std::thread thd{FooCallable(), input}; // ! compiler complains
    thd.join();
    cout << input << endl;
    return 0;
}
```
At first, the variable `input` is value-copied into the buffer inside a thread. Then, when the `FooCallable()` is internally called, the thread attempts to assign an rvalue, namely 100, to a non-const int reference `i`, which is obviously not allowed.

The correct way to pass an argument as a reference is to enclosed it with `std::ref()`. (I personally prefer pointers)
```c++
std::thread thd{FooCallable(), std::ref(input)};
```
Similarly, passing a variable as right value requires the use of `std::move()`. This is particularly useful when an argument is a unique_ptr.
```c++
std::thread thd{FooCallable(), std::move(input)};
```

## std::thread's Copy Control
A std:thread is not copyable but moveable, which enables storing them in a container such as `vector`. A named thread technically needs to be enclosed by std::move() to be returned by a function:

```c++
std::thread createDumbThread() {
    thread t([]() {
        cout << "dumb thread is so dumb" << endl;
    });
    return std::move(t);
}
```
However, you may notice that without the std::move(), the function also compiles:
```c++
std::thread createDumbThread() {
    thread t([]() {
        cout << "dumb thread is so dumb" << endl;
    });
    return t;
}

int main() {
    std::thread t= createDumbThread(); // Output: dumb thread is so dumb
    t.join();
    return 0;
}
```
This is valid thanks to the compiler's Return Value Optimization. [RVO](cpp_primer_read_notes.md#when-a-function-returns-an-instance)

When a thread leaves its scope, its destructor is called, which terminates the thread if it is still `joinable()`, that is, if `join()` or `detach()` has not been called. Thus, moving statement like this `t1 = std::move(t2)` cause the t1's thread to terminate unless t1.detach() or t1.join() was called before.

```c++
thread& operator=(thread&& __t) noexcept
{
    if (joinable())
        std::terminate();
    swap(__t);
    return *this;
}
```

## std::thread::terminate() and stack unwinding
`terminate` is a special function of `std::thread`. It immediately halts the process, killing all threads in it, leaving their stack unwounded. When the process dies, the OS reclaims all process resources:
- Memory (heap, stack, mmap, etc.)
- File descriptors / sockets
- OS-managed handles (threads, semaphores, etc.)

So, from the system’s perspective, nothing leaks permanently.

But…

C++-level cleanup does not happen. Your destructors don’t run in those other threads. For instance,
- If you wrote logs to a file via RAII flush, they may not be flushed.
- If you held a distributed lock in another system (e.g. database, cluster coordinator), the OS reclaiming memory doesn’t notify the other system.

In short, `terminate` ends the process brutally, and it should be prevented.

There are several situations when `terminate` is called. The most likely scenarios are:

- leaving an exception uncaught
- a joinable thread leaves its scope
- a noexcept function throws
- exception inside exception
- others

If the some statement in a thread throws an uncaught exception, for this specific thread, its stack gets unwound before `terminate()` is called. After that, `terminate()` brutally ends the process, leaving all other threads' stack unwounded.

## std::lock() for Locking Multiple Mutex
std::lock() is a variadic function, taking in a few lockable objects and tries to lock them all. It provides all-or-nothing semantics - free previously acquired locks upon failing to access one. Therefore, it's considered deadlock-free.

Take a closer look at the source code: std::lock() locks the first argument and try_lock the rest. Once the try_lock succeeds, that is, `__idx == -1`, the unique_lock frees its control over the first parameter, keeping it locked when function returns. If try_lock fails, another loop begins and the unique_lock, since it is out of scope, would frees the lock it owns. Even when try_lock throws exception, granted lock would be released.
```c++
template<typename _L1, typename _L2, typename... _L3>
void
lock(_L1& __l1, _L2& __l2, _L3&... __l3)
{
    while (true)
    {
        using __try_locker = __try_lock_impl<0, sizeof...(_L3) != 0>;
        unique_lock<_L1> __first(__l1);
        int __idx;
        auto __locks = std::tie(__l2, __l3...);
        __try_locker::__do_try_lock(__locks, __idx);
        if (__idx == -1)
        {
            __first.release();
            return;
        }
    }
}
```
This implementation, nevertheless, is not livelock-free. 

Considering two threads try to access a group of mutexes, but not arrange them in the same order when calling `std::lock()`. Let's say thread 1 acquires the lock1 and thread 2 holds lock2, then both of them would fail in the `do_try_lock` part, resulting in another round of while loop. And the same thing happens again and again..., a classic live lock. So a good practice is that, when we use std::lock(), offer the locks in some fixed order.

One more thing: Usually a std::lock() call is followed by adding lock guards on its input. STL offers a std::scoped_lock class in C++17, which wraps std::lock() in its constructor, freeing us of the fuss.

## Quick Reference Table for Locks & Guards
| Objects | Constructor takes in | Copyable ? | Moveable ? |
|-|-|-|-|
| std::mutex | NA | NO | NO |
| std::lock_guard | reference to a lockable instance | NO | NO |
| std::unique_lock | reference to a lockable instance | NO | YES |
| std::scoped_lock | reference to multiples lockable instances | NO | NO | 
| std::shared_mutex (C++17) | NA | NO | NO
| sdt::shared_lock | reference to a lockable instance | NO | YES |


## std::once_flag and std::call_once
std::call_once is usually used for `lazy-initialization`. It takes in a reference to std::once_flag as the first argument, guaranteeing a callable object is called only once. The implementation is conceptually identical to Golang's `sync.Once`.

```golang
type Once struct {
	done atomic.Uint32
	m    Mutex
}

func (o *Once) Do(f func()) {
	// Note: Here is an incorrect implementation of Do:
	//
	//	if o.done.CompareAndSwap(0, 1) {
	//		f()
	//	}
	// ...

	if o.done.Load() == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done.Load() == 0 {
		defer o.done.Store(1)
		f()
	}
}
```
The atomic flag indicates whether the function is finished(done). When call_once is called, the flag is checked. If it is false, it attempts to obtain an exclusive lock. Naturally, at most one thread would be granted the lock. Such thread executes the callable, set the atomic flag to true when execution ends, and release the lock. Any thread that acquires the lock later on recheck the atomic flag, which is already true, and exits directly.

A digression: after C++, a local static variable's initialization is guaranteed to be thread safe by C++'s implementation.

## Future, Packaged_task and Promise
Since all these three classes employ the same method for synchronization, let's discuss it first.
### _State_baseV2 - A Common Variable
`std::future`, `std::packaged_task`, `std::promise` and `std::shared_future` all directly or indirectly contain a special member variable - a shared pointer of type `_State_base`, which is just a synonym of `_State_baseV2`.
```c++
using _State_base = _State_baseV2;
```
Such state is the core for synchronization - it determines the right action when `std::future::wait()` is called, it synchronizes between a `std::packaged_task` and a `std::future`, it is also the link between `std::promise` with `std::future`. 

`std::future` inherits the state from its parent class `__basic_future`,
```c++
/// Primary template for future.
template<typename _Res>
class future : public __basic_future<_Res> {/*...*/}

template<typename _Res>
class __basic_future : public __future_base {
protected:
    typedef shared_ptr<_State_base>		__state_type;
    typedef __future_base::_Result<_Res>&	__result_type;
private:
    __state_type 		_M_state;
}
```
`std::packaged_task` holds a shard pointer to a derived class of `_State_base`:
```c++
/// packaged_task
template<typename _Res, typename... _ArgTypes>
class packaged_task<_Res(_ArgTypes...)>
{
    typedef __future_base::_Task_state_base<_Res(_ArgTypes...)> _State_type;
    shared_ptr<_State_type>                   _M_state;
    // ...
}

// Holds storage for a packaged_task's result.
template<typename _Res, typename... _Args>
struct __future_base::_Task_state_base<_Res(_Args...)>
: __future_base::_State_base
```
and `std::promise` owns a shared pointer of such state.
```c++
/// Primary template for promise
template<typename _Res>
class promise
{
    typedef __future_base::_State_base 	_State;
    //...
    shared_ptr<_State> _M_future;
}
```
### Synchronization Mechanism of _State_baseV2
Take a look at the state's definition:
```c++
class _State_baseV2 {
    typedef _Ptr<_Result_base> _Ptr_type;

    enum _Status : unsigned {
        __not_ready,
        __ready
    };

    _Ptr_type			_M_result;
    __atomic_futex_unsigned<>	_M_status;
    atomic_flag         	_M_retrieved = ATOMIC_FLAG_INIT;
    once_flag			_M_once;
    
    _Result_base& wait();

    void _M_set_result(function<_Ptr_type()> __res, bool __ignore_failure = false);
}
```
it contains an `__atomic_futex_unsigned`, a struct providing synchronized functionalities that are implemented by system calls(I will not dive deeper, sry). Two functions need our attention, `wait()` and `_M_set_result()`. The former waits for the `_M_status` to be ready and the latter sets it up.

`wait()` is invoked whenever a `future::wait()` or `future::get()` is called. Here is the implementation of `wait()`
```c++
 wait() {
    // Run any deferred function or join any asynchronous thread:
    _M_complete_async();
    // Acquire MO makes sure this synchronizes with the thread that made
    // the future ready.
    _M_status._M_load_when_equal(_Status::__ready, memory_order_acquire);
    return *_M_result;
}

// Wait for completion of async function.
virtual void _M_complete_async() { }
```
Noticed that `_M_complete_async()` is a virtual function, indicating that subclasses of `_State_baseV2` may have their own implementations, as you shall see when we talk about std::async(). The default implementation is empty, doing nothing.

In next line, function waits until the `_M_status` is ready.

`_M_set_result()` is called when `std::promise::set_value()` is called. It sets the state to be ready by calling `_M_status._M_store_notify_all`, a thread safe function.
```c++
// Provide a result to the shared state and make it ready.
// Calls at most once: _M_result = __res();
void _M_set_result(function<_Ptr_type()> __res, bool __ignore_failure = false) {
    bool __did_set = false;
    // all calls to this function are serialized,
    // side-effects of invoking __res only happen once
    call_once(_M_once, &_State_baseV2::_M_do_set, this,std::__addressof(__res), std::__addressof(__did_set));
    if (__did_set)
        // Use release MO to synchronize with observers of the ready state.
        _M_status._M_store_notify_all(_Status::__ready,
                    memory_order_release);
    else if (!__ignore_failure)
        __throw_future_error(int(future_errc::promise_already_satisfied));
}
```
Eventually, by waiting for and setting a futex, thread synchronization is achieved.

### std::async and std::future
`std::async` takes in a launch policy, a function and its arguments, returns a `std::future`, through which we can retrieve the result — the return value of the input function — by calling `get()`.

The launch policy can be either `std::launch::async` or `std::launch::deferred`. The former implies starting a new thread immediately for function execution, while the latter postpones the  execution of the function until `wait()` or `get()` is called. Moreover, `deferred` also means executing the given function in the caller's thread, just as you initialized a lambda and calls it later on. When no policy is given, `std::launch::async` is used(because in the source code, the bits representing `async` is checked before `deferred`). 

```c++
// libstdc++ source code:
/// async, potential overload
template<typename _Fn, typename... _Args>
_GLIBCXX_NODISCARD inline future<__async_result_of<_Fn, _Args...>>
async(_Fn&& __fn, _Args&&... __args)
{
    return std::async(launch::async|launch::deferred,
        std::forward<_Fn>(__fn),
        std::forward<_Args>(__args)...);
}

/// async
template<typename _Fn, typename... _Args>
_GLIBCXX_NODISCARD future<__async_result_of<_Fn, _Args...>> async(launch __policy, _Fn&& __fn, _Args&&... __args)
{
    std::shared_ptr<__future_base::_State_base> __state;
    if ((__policy & launch::async) == launch::async) {
        __try {
            __state = __future_base::_S_make_async_state(
                std::thread::__make_invoker(std::forward<_Fn>(__fn),
                                std::forward<_Args>(__args)...)
            );
        }
    #if __cpp_exceptions
        catch(const system_error& __e) {
            // ...
        }
    #endif
    }
    if (!__state) {
        __state = __future_base::_S_make_deferred_state(
            std::thread::__make_invoker(std::forward<_Fn>(__fn),
                        std::forward<_Args>(__args)...));
    }
    return future<__async_result_of<_Fn, _Args...>>(__state);
}
```
You may have noticed that __state is assigned differently, and this is exactly how the semantics of `async` and `deferred` is implemented. 

`_S_make_async_state` returns a subclass of `_State_baseV2`, and it overrides the `_M_complete_async()` virtual function - simply `join()` the thread created beforehand:
```c++
class __future_base::_Async_state_commonV2
: public __future_base::_State_base {
    // ...
    virtual void _M_complete_async() { _M_join(); }
    void _M_join() { std::call_once(_M_once, &thread::join, &_M_thread); }
    // ...
}
```
Recall that `_M_complete_async()` is indirectly called when `future::get()` is invoked, therefore `future::get()` waits for the thread to finishes.

By comparison, `_S_make_deferred_state` returns another subclass of `_State_baseV2`, which overriding `_M_complete_async()` into:
```c++
template<typename _BoundFn, typename _Res>
class __future_base::_Deferred_state final : public __future_base::_State_base {
    // Run the deferred function.
    virtual void _M_complete_async() {
        // Multiple threads can call a waiting function on the future and
        // reach this point at the same time. The call_once in _M_set_result
        // ensures only the first one run the deferred function, stores the
        // result in _M_result, swaps that with the base _M_result and makes
        // the state ready. Tell _M_set_result to ignore failure so all later
        // calls do nothing.
        _M_set_result(_S_task_setter(_M_result, _M_fn), true);
    }
}
```
It essentially runs the deferred function.

### std::packaged_task
#### basic usage
A `std::packaged_task` is a callable object that wraps any callable target (e.g., functions, lambdas, function object). Its template parameter is a function signature, such as `int(std::string)`, which specifies the return type and argument types.

Because `std::packaged_task` is callable, it can be stored in a `std::function`. When you call its call operator (`operator()`), it executes the wrapped callable and sets the associated result in a `std::future`.

Calling `get_future()` on a `std::packaged_task` returns a `std::future` that will hold the result of the task once it is executed. The future remains in a waiting state until the task is invoked and completes. Calling `wait()` or `get()` on the future will block until the task finishes execution. 

#### Tie Between a Packed Task and its Future
The fact that `std::future` can wait for `std::packaged_task` to be finished suggests some synchronization mechanism between them. It turns out to be a shared pointer of `_State_type`.

```c++
class packaged_task<_Res(_ArgTypes...)> {
    typedef __future_base::_Task_state_base<_Res(_ArgTypes...)> _State_type;
    shared_ptr<_State_type>  _M_state;

    // Result retrieval
    future<_Res> get_future() { 
        return future<_Res>(_M_state); 
    }
}
```
The `future` returned by `get_future` takes a shared pointer of the internal state of the task. As discussed above, `future::get()` waits for such state to be ready and the call operator of task sets the state once the given functions is finished. 

### std::promise
std::promise and std::future usually appear in pairs, the former is a producer and the later is a consumer. Same to `future`, a the template argument of `promise` is the value it wishes to send to the `future`. A `promise` takes in no parameters and returns a `future` when `get_future()` is called. It has two fundamental member functions: `set_value()` and `set_exception()`. Only after either of the function is called then its corresponding `future`'s `get()` or `wait()` returns.

#### Tie Between a Promise and its Future
`std::promise` synchronizes with `std::future` in the same way as `std::packaged_task` does, using a shared pointer that contains a ` __future_base::_State_base`:
```c++
 template<typename _Res>
class promise
{
    typedef __future_base::_State_base 	_State;
    typedef __future_base::_Result<_Res>	_Res_type;
    typedef __future_base::_Ptr<_Res_type>	_Ptr_type;
    template<typename, typename> friend class _State::_Setter;
    friend _State;

    shared_ptr<_State>  _M_future;
    _Ptr_type  _M_storage;
    //...
}
```

### Why is the Internal State a Shared Pointer 
In the destructors of `std::promise` and `std::packaged_task`, you'll find logic like the following:
```c++
if (static_cast<bool>(_M_state) && !_M_state.unique())
        _M_state->_M_break_promise(std::move(_M_state->_M_result));
```
Basically, when a `task` or `promise` leaves its scope, the destructor checks whether its "job" is done — whether the task's call operator is called, or the `promise`'s `set_value` / `set_exception` is called. If not, it sets a `broken_promise` exception and wakes up any threads waiting on the future returned by it.

However, what if no one is waiting? What if the task is created, but `get_future()` is never called? Then there's no need to set such an exception, since nobody cares.

Here comes the clever design — using a shared pointer. Recall that when `get_future()` is called, a `std::future` is constructed, taking the internal state, which is a shared pointer. So the reference count (in the source code it's called `_M_refcount`) increments. Therefore, by checking whether it is the only holder of such a shared pointer, the destructor knows whether to set the exception — this explains the `_M_state.unique()` check in the code!

### get_future() Can Only Be Called Once
Both `std::promise` and `std::packaged_task` return a `std::future` when `get_future()` is invoked. But such invocation can only be done once. Calling it a second time throws a `future_already_retrieved` exception. The mechanism behind is simple - both class has the same `get_future()` implementation. 

```c++
shared_ptr<_State> _M_future;
future<_Res> get_future()
{ return future<_Res>(_M_future); }
```
The function returns a `future` constructed by a shared pointer of _State. The state is then used to construct the base class of `future`, namely, `__basic_future`.

```c++
// Construction of a future by promise::get_future()
explicit
__basic_future(const __state_type& __state) : _M_state(__state) {
    _State_base::_S_check(_M_state);
    _M_state->_M_set_retrieved_flag();
}

void _M_set_retrieved_flag() {
    if (_M_retrieved.test_and_set())
        __throw_future_error(int(future_errc::future_already_retrieved));
}
```
In its constructor, the state is tested and set(an atomic function).


### std::shared_future
Similar to `std::future`, a shared_future is also a subclass of `__basic_future`, but it's both copyable and moveable. A `std::shared_future` can be created by either giving an rvalue of `std::future` or calling `std::future::share()`.

In addition, `get()` can be called multiple times on a certain `shared_future` in that it does not move the result out like `future` does.

```c++
// future:
/// Retrieving the value
_Res get() {
    typename _Base_type::_Reset __reset(*this);
    return std::move(this->_M_get_result()._M_value());
}

// shared_future:
/// Retrieving the value
const _Res& get() const { return this->_M_get_result()._M_value(); }
```

> _"Now, with std::shared_future, member functions on an individual object are still unsynchronized, so to avoid data races when accessing a single object from multiple threads, you must protect accesses with a lock. The preferred way to use it would be to pass a copy of the shared_future object to each thread, so each thread can access its own local shared_future object safely, as the internals are now correctly synchronized by the library. Accessing the shared asynchronous state from multiple threads is safe if each thread accesses that state through its own std::shared_future object."_


## Quick Reference Thread Related Templates
| Objects | template type | template argument| Constructor takes in | Copyable ? | Moveable ? |
|-|-|-|-|-|-|
| std::future | class | type of return value | Usually Not Constructed Directly | NO | YES |
| std::shared_future | class | type of return value | std::future&& | YES | YES |
| std::async | function | function signature | callable and arguments | - | - | 
| std::packaged_task | class | function signature | callable | NO | YES|
| std::promise | class | type of return value | NA | NO | YES |

## Atomic Variable
### Compare Exchange Weak & Strong
There are two member functions of `std::atomic<T>`, `compare_exchange_weak` and `compare_exchange_strong`. On a platform where compare and swap is an atomic instruction, such as x86, these two functions have exactly the same effect - check if the current atomic owns a certain value, if so, modify it into a different one; otherwise, store the current value into the reference. Nevertheless, on a platform such as arm64, where compare and swap is implemented in multiple instructions, `compare_exchange_weak` may fail even when expected value == the atomic. At such times, `compare_exchange_strong` retries the assignment continuously when there is a spurious failure. Conceptually, it acts as if:

```c++
template<typename T>
bool compare_exchange_strong(std::atomic<T>& atomic_obj, T& expected, T desired) {
    T original = expected;
    while (!atomic_obj.compare_exchange_weak(expected, desired)) {
        if (expected != original) {
            // Real mismatch: atomic value changed by another thread
            break;
        }
        // Spurious failure: retry
    }
    return expected == original;
}
```
### User Defined Atomic Type
#### Restrictions
std::atomic<T> templates enables us to create atomic version of user defined types. However, such template imposes stringent requirements for the underlying struct/class - it must be trivially copyable and it must overload operator `==` and `!=`( compiler for c++20 automatically generate them if not defined ).

Trivially copyable, as its name suggests, means that a new object of that class can be created by copying the raw bytes of an existing one using only `memcpy`. It implies that for a class:

1. All the copy/move control members, namely copy constructor, copy operator, move constructor, move operator, destructor, must be compiler synthesized, or marked "= default".
2. Does not contain any self-defined or inherited `virtual` functions.
3. Neither the type nor any of its base classes may use `virtual` inheritance.

The rationale behind is simple: the template regards a user defined type as a piece of continuous memory(raw bytes).

#### Lock Free Criteria
Since compiler treats `T` in `atomic<T>` as a block of raw bytes, if `sizeof(T)` is larger than the word size, a mutex needs to be used. As seen above, compare and exchange is usually implemented by a single machine instruction which accepts at most word-sized memory. If sizeof(T) is larger than word size or misaligned across word boundaries, such instruction must be called multiple times. To protect the data from being modified by another thread in between these calls, a mutex will be needed, making `atomic<T>` NOT lock free.

When a `atomic<T>` is not lock free, since we’re adding a mutex anyway, why not manage it ourselves and relax the trivially copyable requirements? Well, it is trade-off: use library provided atomic avoids incorrect memory order problem and the code is less error pruning, while manage mutex ourselves gives us the more freedom, perhaps finer granularity.

### Memory Ordering
It's really an intricate topic.

`Memory order` is a tool designed to manage and coordinate:
- Compiler-level reordering: The compiler optimizes by reordering instructions unless told not to.
- CPU-level out-of-order execution: Modern CPUs execute instructions out-of-order to maximize throughput, so the hardware can reorder memory operations dynamically.
- CPU cache coherence and memory visibility: Multiple cores with local caches need to keep data consistent and propagate updates in a way that respects synchronization.

In reality, the surprising or “mysterious” behaviors you sometimes see in concurrent programs often results from the combination of these three.

#### memory_order::relaxed 
This order applies no restriction on instructions ordering and CPU caching.

Let's say we have an atomic variable named `x`, whose reference is shared among a few concurrent threads.

`x.store(..., relaxed)` updates the corresponding cache entry in CPU it runs on, so that subsequent `x.store(..., load)` reads a version as new as this one. The store action also asks other CPUs to update their caches for `x`, but it happens at some unknown future time. As a result, a relaxed load on another thread running on a different core may possibly read a staled value in that only the CPU cache is consulted.

For a single atomic variable, once a thread observes a particular version (value), it will only see that version or newer ones going forward — because the cache will only be updated with newer versions, never older ones. But for multiple atomic variables, each one’s updates propagate independently. So a thread may see different variables at different "versions," leading to inconsistent or unsynchronized observations across variables.

#### memory_order::acquire and release
When one thread does a `release‑store` (or successful release‑CAS) to atomic `x`, and another thread subsequently does an `acquire‑load` (or successful acquire‑CAS) of `x`, the second thread is guaranteed to see all the writes that happened before the release in the first thread.

Instruction‑reordering angle:

`Release` acts like a write‑fence:
It prevents the compiler or CPU from moving any memory access that appears before the release goes below it.

`Acquire` acts like a read‑fence:
It prevents any memory access appearing after it from being moved above it.

#### RMW Operation
When a thread performs an `RMW` (Read Modify Write) such as fetch_sub and compare exchange, the CPU must gain exclusive ownership of the cache line that holds x.

This involves:

- Consulting other CPUs (via the cache coherence protocol, e.g., MESI or MOESI).
- Asking them to invalidate or update their cached copies of that cache line.
- Only after confirming it has the latest value (the most up-to-date version of x), can it safely read, modify, and write back the new value atomically.

This arises a problem: if RMW always reads the newest version, calling it with `memory_order::acquire` seems meaningless. However, this understanding is incorrect, because the `memory_order::acquire` in an RMW is not for the RMW itself, but for what follows it: 

> Atomic RMW ops (like compare_exchange) read the latest value from memory (due to coherence), but do not enforce visibility of other memory unless acquire or release order is used.

It doesn't affect the atomicity of the operation, which is always guaranteed, or how up-to-date the atomic variable's value is (that's guaranteed by cache coherence and the RMW mechanism). But it does influence the visibility of other variables written by another thread before a store(..., memory_order::release).

```c++
std::atomic<int> ready = 0;
Data data;

// Thread A
data.fill();                                      // initialize shared data
ready.store(1, std::memory_order_release);        // publish visibility

// Thread B
if (ready.fetch_sub(1, std::memory_order_acquire) > 0) {
    use(data);                                    // safe: sees published writes
}
```
In the example, thread B may not have been able to see the filled data had the `fetch_sub` takes in a memory_order_relaxed. BTW, the release-store and RMW forms a classic `release sequence`

## Synchronized Data Structure With Locks
### Defer Expensive Cleanup Outside Critical Sections
Let’s say you need to change a pointer that’s protected by a mutex:
```c++
class Bar {
    ~Bar() {
        // some expensive operation
    }
};

class Foo {
    std::mutex mtx_;
    Bar* ptr_;
    void reset_ptr(Bar* new_addr) {
        std::lock_guard<std::mutex> guard(mtx_);
        delete ptr_;
        ptr_ = new_addr;
    }
};
```
The code above works, but note that `delete ptr_` occurs while the mutex is locked. Since Bar's destructor may be expensive, it's good practice to perform such cleanup outside the critical section. A simple improvement would be:

```c++
void reset_ptr(Bar* new_addr) {
    Bar* old_ptr;
    {
        std::lock_guard<std::mutex> guard(mtx_);
        old_ptr = ptr_;
        ptr_ = new_addr;
    }
    delete old_ptr;
}
```
This is straightforward. But if Foo stores the pointer using a smart pointer like std::shared_ptr, the issue becomes subtler:
```c++
class Foo {
    std::mutex mtx_;
    std::shared_ptr<Bar> ptr_;
    void reset_ptr(Bar* new_addr) {
        std::lock_guard<std::mutex> guard(mtx_);
        ptr_ = shared_ptr<Bar>(new_addr);
    }
};
```
Here, assigning a new `shared_ptr` to `ptr_` causes the old one’s reference count to decrement. If `ptr_` was the last reference, this reassignment destroys the `Bar` object inside the critical section — which again is not ideal for high-concurrency scenarios.

A better approach is to defer the destruction:
```c++
void reset_ptr(Bar* new_addr) {
    std::shared_ptr<Bar> defer;
    {
        std::lock_guard<std::mutex> guard(mtx_);
        defer = std::move(ptr_);
        ptr_ = shared_ptr<Bar>(new_addr);  
    }
    // `defer` goes out of scope here, after the lock is released
}
```
By moving the old `shared_ptr` to a local variable, we delay its destruction until after the mutex is released. If this was the last reference, the actual destructor of Bar runs after the critical section.

## A Digression: Questions to Ask When Designing a Class
I often ask myself the following questions while implementing a new class:
- Must I favor raw pointers over smart pointers?
- When creating objects on heap and using raw pointers, can I guarantee that the following code is `noexcept`? Otherwise, there is a risk of memory leak should some statement throws.
- Is my function exception-safe? Can I guarantee no loss of data when mistakes occur?
- Can I move instance creation out of critical region?
- Can I postpone the destructor call of an object until lock is released?

## Lock-free Synchronized Data Structure
### Memory Reclamation Strategy: Two Reference Counter
```c++
//
// Created by Charles Green on 7/2/25.
//

#ifndef MYLOCKFREESTACK3_H
#define MYLOCKFREESTACK3_H

// Warining: only works for architecture supporting canonical addressing
template <typename T>
class MyLockFreeStack3 {
    struct StackNode;
    // This following CountedPointer would make atomic<CountedPointer> NOT lock-free.
    // But fortunately, on x64 machine, beacuse of canonical address, a pointer in fact uses only the lower 48 bits,
    // so we can use the extra 16 bits to store the acquire_cnt_. This magic is called pointer tagging.
    // struct CountedPointer {
    //     int16_t acquire_cnt_;
    //     StackNode* node_;
    // };
    // The following CountedPointer is simply a wrapper of uint64_t, thus, atomic<CountedPointer> is lock free
    class CountedPointer {
        uint64_t internal_representation_;
        constexpr static uint64_t ADDR_MASK = (1ULL << 48) - 1;
        constexpr static uint64_t packIntoCountedPointer(StackNode* addr, int16_t rf) {
            uint64_t ret = 0;
            assert((reinterpret_cast<uint64_t>(addr) & ~ADDR_MASK) == 0);
            ret |= ADDR_MASK & reinterpret_cast<uint64_t>(addr);
            ret |= static_cast<uint64_t>(rf) << 48;
            return ret;
        }
    public:
        CountedPointer(StackNode* ptr, uint16_t rf): internal_representation_(packIntoCountedPointer(ptr, rf)) {}
        CountedPointer(): internal_representation_(0) {}
        bool operator==(const CountedPointer& another) const {
            return internal_representation_ == another.internal_representation_;
        }
        bool operator!=(const CountedPointer& another) const {
            return internal_representation_ != another.internal_representation_;
        }
        StackNode* pointer() const {
            return reinterpret_cast<StackNode*>(internal_representation_ & ADDR_MASK);
        }

        int16_t counter() const {
            return static_cast<int16_t>(internal_representation_ >> 48);
        }
    };

    struct StackNode {
        std::atomic<int16_t> release_cnt_;
        std::shared_ptr<T> data_;
        CountedPointer next_;
        StackNode(): release_cnt_(0) {}
    };
    // when the stack is emtpty, there is still a dummy node at head_. The acquire_cnt_ of the dummy node may be any
    // positive interger.
    std::atomic<CountedPointer> head_;
public:
    MyLockFreeStack3() = default;
    template <typename U>
    void push(U&& input) {
        StackNode* sn = new StackNode();
        sn->data_ = std::make_shared<T>(std::forward<U>(input));
        sn->next_ = head_.load();
        CountedPointer ncp(sn, 0);
        while(!head_.compare_exchange_weak(sn->next_, ncp));
    }

    CountedPointer get_and_increase() {
        CountedPointer old_cp = head_.load();
        CountedPointer new_cp;
        int16_t counter = 0;
        StackNode* ptr;
        do {
            new_cp = old_cp;
            ptr = old_cp.pointer();
            counter = old_cp.counter();
            new_cp = CountedPointer(ptr, counter + 1);
        } while(!head_.compare_exchange_strong(old_cp, new_cp));
        return new_cp;
    }

    std::shared_ptr<T> pop() {
        while(true) {
            CountedPointer head_cp = get_and_increase();
            StackNode* ptr = head_cp.pointer();
            int16_t ac_cnt = head_cp.counter();
            if(!ptr) {
                // we don't care about the dummy node's acquire_cnt_
                return nullptr;
            }
            if(head_.compare_exchange_strong(head_cp, ptr->next_)) {
                // For a certain node, only the thread owning the largest acquire_cnt_ can pass the above compare exchange strong.
                // The reason is simple: suppose by contradition that there is another thread, X, holding
                // this node but with a larger acquired_cnt_. To obtain this node, thread X must have finished calling get_and_increase().
                // And that means the `head_` must have been updated to X's version - a CountedPointer with larger acquire_cnt_.
                // But that would make the current compare_exchange_strong fail - a contradiction.
                std::shared_ptr<T> ret = std::move(ptr->data_);
                // there are head_cp.acquire_cnt_ threads used to hold this node, so there should be head_cp.acquire_cnt_ fetch_add(1)
                int addon = - ac_cnt + 1;
                if(ptr->release_cnt_.fetch_add(addon) == -addon) {
                    delete ptr;
                }
                return ret;
            }
            if(ptr->release_cnt_.fetch_add(1) == -1) {
                delete ptr;
            }
        }
    }
};


#endif //MYLOCKFREESTACK3_H

```

## Exception Safe Concern
If a thread throws an uncaught exception, the entire program will terminate — a situation that should be avoided. Similarly, if the main thread throws an exception before it calls join() or detach() on a child thread, the program may terminate due to undefined behavior. These issues can be gracefully handled using std::async or std::packaged_task.

When using std::async or std::packaged_task, and storing the associated std::future, any exception thrown in the thread is captured in the future’s internal state. The exception is then rethrown when `.get()` is called. In the case of `std::async`, if used with the std::launch::async policy, the function starts executing in a new thread immediately. This thread is automatically joined when the shared internal state is destroyed, which prevents termination due to an unjoined thread if the main thread exits early.