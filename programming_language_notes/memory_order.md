# Memory Order in Golang And C++
## happens-after relationship
1. the creation of a std::thread or goroutine happens before the execution of it. 
```c++
int main() {
    printf("123\n");
    std::thread t([]() {
        printf("789\n");
    });
    t.join();
    return 0;
}
// the result is always: 
// 123
// 789
```
```golang
var x, y int
func f1() {
    x, y = 123, 789
    go func() {
        fmt.Println(x)
        go func() {
            fmt.Println(y)
        }()
    }()
}
```

2. channel operation:
- The n-th non-blocking channel send happens before n-th channel receive completes
- If a receive wakes up a blocked send, then the receive happens before the send completes
- Channel closing operation happens before receiving a zero value(because of channel closing) from it.