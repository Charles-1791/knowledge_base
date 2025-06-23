# C++ Concurrency in Action Read Notes
## Arguments of std::thread
### Single Argument
std::thread's constructor takes in any callable object, namely:
- Function pointer
- Lambda
- Functor Object(a class with operator())
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

- When we create an instance of std::thread by calling its constructor, the constructor blindly value-copies all arguments into its own buffer. 

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
### std::future
#### std::async and std::future
`std::async` takes in a launch policy, a function, and its arguments. It returns a `std::future`, through which we can retrieve the result — the return value of the input function — by calling `get()`.

The launch policy can be either `std::launch::async` or `std::launch::deferred`. The former implies starting a new thread immediately for function execution, while the latter postpones the  execution of the function until `wait()` or `get()` is called. Moreover, `deferred` also means executing the given function in the caller's thread, just as you initialized a lambda and calls it later on. 

When no policy is given, `std::launch::async` is used. 

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
_GLIBCXX_NODISCARD future<__async_result_of<_Fn, _Args...>>
async(launch __policy, _Fn&& __fn, _Args&&... __args)
{
    std::shared_ptr<__future_base::_State_base> __state;
    if ((__policy & launch::async) == launch::async)
    {
        __try
        {
            __state = __future_base::_S_make_async_state(
            std::thread::__make_invoker(std::forward<_Fn>(__fn),
                            std::forward<_Args>(__args)...)
            );
        }
        // ...
    }
```
This can also be verified by a simple experiment:
```c++
int main() {
    std::future<void> ft = std::async( []() {
        cout << "std::function thread id: " << std::this_thread::get_id() << endl; // Output: std::function thread id: 140204404901632
    });
    cout << "main thread's id: " <<  std::this_thread::get_id() << endl; // Output:: main thread's id: 140204405569344
    ft.get();
    return 0;
}
```

For the returned `future` object, `get()` can only be called once, as it moves the internally stored value out. After `get()` is called, the future becomes invalid and cannot be used again.
```c++
// libstdc++ source code:

/// Retrieving the value
_Res get()
{
    typename _Base_type::_Reset __reset(*this);
    return std::move(this->_M_get_result()._M_value());
}
```

#### Diving Deeper Into std::future
`std::future` inherits from `__basic_future`, whose constructors are all `protected`. 
```c++
// stdlibc++ source code
template<typename _Res>
class __basic_future : public __future_base
{
private:
    __state_type 		_M_state;
protected:
    // ...
    explicit
    __basic_future(const __state_type& __state) : _M_state(__state)
    {
    _State_base::_S_check(_M_state);
    _M_state->_M_set_retrieved_flag();
    }

    // Copy construction from a shared_future
    explicit
    __basic_future(const shared_future<_Res>&) noexcept;

    // Move construction from a shared_future
    explicit
    __basic_future(shared_future<_Res>&&) noexcept;

    // Move construction from a future
    explicit
    __basic_future(future<_Res>&&) noexcept;

    constexpr __basic_future() noexcept : _M_state() { }

    // ...
};

```
Internally, a `__basic_future` maintains a state named `_M_state`, which determines the right action when `wait()` is called. Before returning the result, a state always call a function `_M_complete_async()`.
```c++
// stdlibc++ source code
class _State_baseV2 {
    // ...
   _Result_base&
    wait()
    {
        // Run any deferred function or join any asynchronous thread:
        _M_complete_async();
        // Acquire MO makes sure this synchronizes with the thread that made
        // the future ready.
        _M_status._M_load_when_equal(_Status::__ready, memory_order_acquire);
        return *_M_result;
    } 
    // ...
    // Wait for completion of async function.
    virtual void _M_complete_async() { }
}

```
Noticed that `_M_complete_async()` is a virtual function, indicating that states inherits from `_State_baseV2` may have their own implementations. If a state fails to override this function, a `wait()` call would hang until the result is ready, exactly the case for the `future` returned by calling `get_future()` on `std::packaged_task`. 

In fact, this function is overridden by two derived class templates : `_Deferred_state` and `_Async_state_commonV2`, corresponding to `deferred` and `async`, the two policies for `std::future`.
```c++
// libstdc++ source code:
using _State_base = _State_baseV2;
// ...
// Shared state created by std::async().
// Holds a deferred function and storage for its result.
template<typename _BoundFn, typename _Res>
class __future_base::_Deferred_state final
: public __future_base::_State_base {
    // ...
    virtual void _M_complete_async()
    {
        _M_set_result(_S_task_setter(_M_result, _M_fn), true);
    }
    // ...
}

// ...

// Common functionality hoisted out of the _Async_state_impl template.
class __future_base::_Async_state_commonV2
: public __future_base::_State_base {
    // ...
    virtual void _M_complete_async() { _M_join(); }
    // ...
}
```
Apparently, only when wait() is called does a `deferred` state executes the function. By comparison, `async` just `join()` the thread it created beforehand.

### std::packaged_task
#### basic usage
A `std::packaged_task` is a callable object that wraps any callable target (e.g., functions, lambdas, functors). Its template parameter is a function signature, such as `int(std::string)`, which specifies the return type and argument types.

Because `std::packaged_task` is callable, it can be stored in a `std::function`. When you call its call operator (`operator()`), it executes the wrapped callable and sets the associated result in a `std::future`.

Calling `get_future()` on a `std::packaged_task` returns a `std::future` that will hold the result of the task once it is executed. The future remains in a waiting state until the task is invoked and completes. Calling `wait()` or `get()` on the future will block until the task finishes execution.  

#### Tie Between a Packed Task and its Future
The fact that future can wait for task to be finished suggests some synchronization mechanism between them. It turns out to be a shared pointer of `_State_type`.

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
The `future` returned by `get_future` takes a shared pointer of the internal state of the task. `wait()` waits for such state to be ready and the call operator of task sets certain bits in it once the given functions is finished. 

Take a closer look at the state, it is of type `_Task_state_base`, which is a subclass of `_State_base`, aka `_State_baseV2`.
```c++
using _State_base = _State_baseV2;

template<typename _Res, typename... _Args>
struct __future_base::_Task_state_base<_Res(_Args...)>
: __future_base::_State_base
```
In `_State_baseV2`, the member truly being watched and modified is `_M_status`, an `atomic futex`. Setting and waiting for such variable are done by atomic operations(syscalls), and are therefore guaranteed to be thread-safe. 
```c++
class _State_baseV2
{
    typedef _Ptr<_Result_base> _Ptr_type;
    // ...
    _Ptr_type			_M_result;
    __atomic_futex_unsigned<>	_M_status;
    // ...
}
```

#### When the Call Operator is Never Invoked...
When a future is out of scope, its destructor checks if the call operator has been called. If no, it stores a `broken_promise` exception into the internal state and immediately wakes up any thread waiting on the `future`:
```c++
~packaged_task() {
    if (static_cast<bool>(_M_state) && !_M_state.unique())
        _M_state->_M_break_promise(std::move(_M_state->_M_result));
}
```

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
#### When Neither Value nor Exception is set
Identical to packaged_task, its destructor sets a `broken_promise` exception when neither of set_value or set_exception is invoked.
```c++
~promise() {
    if (static_cast<bool>(_M_future) && !_M_future.unique())
        _M_future->_M_break_promise(std::move(_M_storage));
}
```

## Quick Reference Thread Related Templates
| Objects | template type | template argument| Constructor takes in | Copyable ? | Moveable ? |
|-|-|-|-|-|-|
| std::future | class | type of return value | Usually Not Constructed Directly | NO | YES |
| std::async | function | function signature | callable and arguments | - | - | 
| std::packaged_task | class | function signature | callable | NO | YES|
| std::promise | class | type of return value | NA | NO | YES |