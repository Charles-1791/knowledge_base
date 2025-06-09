# C++ Notes
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
    int extra;
public:
    Lambda(int extra): extra(extra){} // a bad practice, only for illustration
    int operator()(int a, int b) {
        ++extra;
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
All member functions are compiled into machine instructions and stored at fixed addresses, specifically a read-only area called `.text`. The rationale is simple, enable sharing among instances of the same class.

#### Member variable
A non-static member variable is stored in the instance itself, so it can be on stack or heap.

#### Non-Const Static Member 
If initialized, it is stored in the `.data` segment, otherwise, it is initialized in `.bss` segment.

#### Const Static Member
This is the tricky one. After `C++11` compilers can optimize a `const static` member into a `constexpr` so that it directly replace all its occurrence with the value literal, avoiding allocating memory for it. However, before `c++`, when there is no such optimization, such a variable is stored in `.rodata`.

#### Vptr & Virtual Table
If a class has at least one virtual function, then the compiler generates a hidden field called `vptr` inside each instance, which points to the `vtable`, a table storing all virtual functions' address. The virtual table itself(generated at compile time), like the member functions, is shared among all instances and is therefore stored in `.rodata`.

### Function Call
#### Non-Virtual Function Call
The compiler translates these calls into calls to a fixed address because the function’s address is known at compile time.

As mentioned earlier, the machine instructions for member functions are shared among all instances. Therefore, the call implicitly passes the this pointer as a hidden argument to operate on the specific instance.

#### Virtual Function Call
A virtual function call must be triggered through a reference or pointer. It resembles calling a method on an interface in Go.

At runtime, the `vptr` (virtual table pointer) stored in the concrete instance is first dereferenced to access the `vtable`. Since function names do not exist at the machine code level — they are replaced by offsets during compilation — the call uses a specific offset into the `vtable` to locate the actual function address. The function at that offset is then invoked.

#### Devirtualization

By comparison, if you call a function via an instance(not reference, not pointer), then even if it is declared as virtual, such call won't use `vptr` and `vtable` because the address is known at compile time. This compiler optimization is commonly known as `Devirtualization`.Here is an example:

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
