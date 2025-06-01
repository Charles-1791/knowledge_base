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

