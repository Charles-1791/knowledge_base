# C++ Primer Read Notes
## When a function returns a function pointer ...
### Variables captured by reference
We have the following scenario:
```c++
std::function<int(int, int)> getFunction() {
    int extra = int(100);
    return [&](int a, int b)->int {
        ++extra;
        return a + extra + b;
    };
}
```
Since the *extra* is reference copied, after the getFunction returns, the *extra* is popped out of the stack. Calling the returned function results in *Undefined Behavior*.

### Variables captured by value
Here we chang the & into =, and add a *mutable* after parameter list.
```c++
std::function<int(int, int)> getFunction() {
    int extra = int(100);
    return [=](int a, int b) mutable ->int {
        ++extra;
        return a + extra + b;
    };
}
```
The returned function is now safe to call because lambda has saved a copy of the *extra* in itself. 

We could consider a lambda like a callable object like this:

```c++
class Lambda {
private:
    int extra_;
public:
    Lambda(int extra): extra_(extra){}
    int operator()(int a, int b) {
        ++extra_;
        return a + extra + b;
    }
};
```

One more thing: we must add a keyword *mutable* after the parameter list should we change *extra* inside the function body. This is because by default, lambda treat the call operator, that is () operator, a const function. We can interpret it like this:

```c++
int operator()(int a, int b) const {
    ++extra_; // compiler complains here
    return a + extra_ + b;
}
```

## About Class/Struct
### Memory Layout
#### Member functions
All member functions are compiled into machine instructions and stored at fixed addresses, typically in a read-only section called `.text`. This design allows instances of the same class to share the same function code, avoiding duplication and improving memory efficiency.

#### Member variable
Since a non-static member variable is stored in the instance itself, it can be on stack or heap.

#### Non-Const Static Member 
If initialized, it is stored in the `.data` segment, otherwise, it is initialized in `.bss` segment.

#### Const Static Member
This is the tricky one. After `C++11` compilers can optimize a `const static` member into a `constexpr` so that it directly replace all its occurrence with the value literal, avoiding allocating memory for it. However, before `c++`, when there is no such optimization, such a variable is stored in `.rodata`.

#### Vptr & Virtual Table
If a class has at least one virtual function, then the compiler generates a hidden field called `vptr` inside each instance, which points to the `vtable`, a table storing all virtual functions' address. The virtual table itself(generated at compile time), like the member functions, is shared among all instances and is therefore stored in `.rodata`.

### Function Call
#### Non-Virtual Function Call
The compiler translates member function calls into calls to fixed addresses, as the function's location is known at compile time.

As mentioned earlier, the machine instructions for member functions are shared across all instances of a class. To allow these shared functions to operate on specific instances, the compiler implicitly passes the `this` pointer as a hidden argument with each call.

#### Virtual Function Call
A virtual function call must be triggered through a reference or pointer. It resembles calling a method on an interface in Go.

At runtime, the `vptr` (virtual table pointer) stored in the concrete instance is first dereferenced to access the `vtable`. Since function names do not exist at the machine code level — they are replaced by offsets during compilation — the call uses a specific offset into the `vtable` to locate the actual function address. The function at that offset is then invoked.

#### Devirtualization
By comparison, if you call a function via a concrete instance(not reference, not pointer), then even if it is declared as virtual, such call won't use `vptr` and `vtable` because the address is known at compile time. This compiler optimization is commonly known as `Devirtualization`.Here is an example:

```c++
struct Foo {
    virtual void hello() {}
};

struct Bar: public Foo {
    void hello() override {} 
};

int main() {
    Bar* bar = new Bar();
    bar->hello(); // Devirtualization
  	return 0;
}
```
### When a Function Returns an Instance...
Sometimes, a function creates an instance of some class and returns it. The returned instance is then assigned to a variable.
```c++
struct Foo {
    int a;
    int b;
    Foo() { std::cout << "default ctor\n"; }
    Foo(const Foo&) { std::cout << "copy ctor\n"; }
    Foo(Foo&&) { std::cout << "move ctor\n"; }
};

Foo getFoo() {
    Foo f;
    f.a = 1;
    f.b = 1;
    return f;
}


int main() {
    Foo tmp = getFoo();
    return 0;
}
```
The process involves three copy-control function calls. In `getFoo`, when `f` is created, the `default constructor` is invoked. When `getFoo` returns `f`, which is going to be popped out of stack, the `move constructor` is called. Eventually, when the return value is used to initialize `tmp`, another `move constructor` is called. (If Foo has no move constructor, the copy constructor would be called twice instead).

However, if you compile and run the code above, you would observe that only the `default constructor` is called. That's because the compiler did a `RVO (Return Value Optimization)`, which lets the compiler construct the return value directly in the memory location of the caller, skipping the need for a copy or move.

To disable RVO and observe the exact copy control calls, do this:

```bash
$ g++ -fno-elide-constructors main.cpp -o main
$ ./main
```
You shall see:
```
default ctor
move ctor
move ctor
```

## Secrets Behind delete[]
When we dynamically allocate an array with `T* arr = new T[sz]`, we need to recycle it after use by calling `delete []arr`. Since the arr only carries the starting address of the first element, the sz is not recorded, then how can the runtime know how much space to reclaim? 

Before we offer the explanation, a notion needs to be clarified - `trivially destructible`. Similar to `trivially copyable`, a `trivially destructible` type indicates there is no customized destructor defined for this type. In other words, there is no extra work to be done when the variable of this type is out of scope like free memory on heap or close internet connection. For instance, `int`, `double`, `void*` has no destructor, so they are `trivially destructible`. 

Now let's solve the puzzle. When `new` is called, the heap allocator always its own hidden bookkeeping metadata, which contains the allocated memory size. 

- If the array elements are trivially destructible, then there is no need to know the exactly number of elements inside the array and the cleanup job is just a simple `free(mem_size)`.
- If elements are not trivially destructible, destructors must be called for each element, necessitating the knowledge of element count. stdlibc++ achieves so by secretly storing a `size_t` before the first element. The following experiment unravels its existence.

```c++
struct Foo {
    int* internal_ptr;
    Foo(): internal_ptr(new int(0)) {}
    ~Foo() {
        delete internal_ptr;
    }
};

int main() {
    Foo* arr = new Foo[260];
    size_t sz = *(reinterpret_cast<size_t*>(arr) - 1);
    cout << sz << endl; // output: 260
    delete[] arr;
    return 0;
}
```

## About Multiple Definition
In C++, a `class` or `struct` is often referenced by multiple `.cpp` source files. To avoid duplicating code in each file, we use header files (`.h`) and the `#include` directive. The `#include` directive essentially copies the contents of a header file into the current file during preprocessing.

However, C++ does not allow multiple definitions of the same object across translation units. Since including a header file in multiple `.cpp` files results in its contents being duplicated, header files should generally contain only declarations, not definitions. Function and class declarations locate in header files, while their implementations are placed in corresponding `.cpp` files.

### Global Variables and Extern
To share a global variable across multiple `.cpp` files, we must declare it in a header file. But unlike functions or classes, a variable declaration in C++ is also a definition unless explicitly marked otherwise. To resolve this, we use the **extern** keyword to declare the variable in the header file and then define and initialize the variable in exactly one `.cpp` file:

```c++
// in xxx.h:
extern int shared_int;

// in xxx.cpp:
int shared_int = 114514;
```

### Static Class Members
Similarly, a `static` data member of a class must be defined exactly once, usually in a `.cpp` file:

```c++
// in xxx.h
class Foo {
    static int shared_member;
};

// in xxx.cpp
int Foo::shared_member = 1919810;
```

### Inline Variables and Functions (C++17 and Later)
C++17 introduced the inline keyword for variables, allowing both declaration and definition in header files. This avoids linker errors due to multiple definitions:

C++17 introduces a syntax sugar: `inline`. By adding `inline` before a global variable or static member, we can add its definition in the head file:

```c++
// in xxx.h:
inline int shared_int = 114514;
class Foo {
    inline static int shared_int = 1919810;
}
```
Introducing `inline` also allows defining free functions directly in header files - before C++17, we can only write function definition inside `.cpp` files. 

```c++
in xxx.h
inline void foo_func(int input) {
    ...
}
```

### Const Global Variables
A const global variable defined in a header file does not trigger a multiple definition error, even when not marked extern, because in C++, const variables have internal linkage by default. This behavior is similar to a static global variable in a header: each translation unit (i.e., each .cpp file that includes the header) gets its own separate copy of the variable.

In other words, the variable is not shared across files, but rather duplicated into each compilation unit. If you need a shard const global variable, you need to separate its declaration and definition, as we do to normal global variable.


### Implicit Inline Marks
You may wonder: since definition should not appear in a head file unless marked `inline`, why can we define some member functions inside a class definition, which usually resides in a head file. 

That is because member functions defined inside a class are implicitly inline. Notes: `inline` also suggests compiler to inline the function during compilation, similar to a MACRO, but the compiler decides whether to actually do it.

### A Digression: ifndef and Multiple Definitions
Another common question is whether macros like `#ifndef`, `#define`, and `#endif` solve the multiple definition problem. The answer is: not directly. 

These are include guards, which prevent the same header file from being included multiple times in a single translation unit. For example, if file A includes header B, and file C includes both A and B, include guards prevent two copies of B’s code from being inserted. This avoids compilation errors due to **duplicate declarations**, but does not solve the **multiple definition** problem, which occurs across multiple translation units at link time.
