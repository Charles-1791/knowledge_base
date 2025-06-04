# Golang Notes

## compilation
- "go run xxx.go" compiles a single input and execute it

- "go build xxx.go" creates an executable binary

- func init() {} is a special function automatically called when the program are executed. It can not be called in the code.

## syntax
- a++ is a statement, not a expression, so b = a++ is invalid

## Golang type categories

### basic types
- int, int8, int16, int32, int64
- uint, uint8, uint16, uint32, uint64
- float32, float64 (there is no float)
- complex64, complex128
- bool
- string

### aggregate types
- array
- struct

### reference types

consider them as pointers 
- pointer
- slice (not comparable)
- map (not comparable)
- function (not comparable)
- channel (not comparable)

### interface types
- interface

## string, []byte and []rune

String is immutable. To change a character, we need to convert string into []rune on which we apply the modification and then convert it back.

Obeying the UTF-8 standard, a string may contain characters made up of multiple bytes, so the right approach to iterate the string is to convert it into []rune first, and go through it.

a for-loop iterates a string rune by rune rather than byte by byte.

Converting a string into a []rune allocates new memory. Since rune is fixed-lengthed, the size of the converted []rune is always no less than that of string.

unsafe.Sizeof(x) can get the size of a variable, but if x is string, it returns its head size, that is 16 on a 64-bit system, we can consider a string in golang having the following structure:
```golang
type string struct {
    Data *byte  // pointer to the actual UTF-8 bytes
    Len  int    // length in bytes
}
```

Converting a string to []byte also necessitates memory allocation, and so does the reverse.

## array and slice

### slice
- unsafe.Sizeof() returns a slice's head size, which is 24 on a 64-bit system. We may consider a slice like this:
```golang
type slice struct {
    Data *T  // pointer to the underlying array
    Len  int // number of elements
    Cap  int // capacity of the array
}
```
- We can use T[]{idx1: ele1, idx2: ele2} to initialize a slice

- cap(slc) and return the capacity of the slice

#### Memory Concern

- Using a subscript operator, that is X[i:j], to generate a sub-slice Y from some slice X actually allocates no memory. Mutation to Y would affect X unless Y points to a new piece of chunk, which happens when appending a new element after the capacity is exhausted. 

```golang
	X := []int{0,1,2,3,4,5,6,7,8}
	fmt.Println(cap(X))
	// output: 9
	Y := X[3:5]
	Y[0] = 300
	Y = append(Y, 500)
	Y = append(Y, 600)
	Y = append(Y, 700)
	Y = append(Y, 800)
	fmt.Printf("X = %v\n", X)
	// output: X = [0 1 2 300 4 500 600 700 800]
	fmt.Printf("Y = %v\n", Y)
	// output: Y = [300 4 500 600 700 800]
	// now, Y points to a different piece of memory
	Y = append(Y, 900)
	Y[1] = 400
	fmt.Printf("X = %v\n", X)
	// output: X = [0 1 2 300 4 500 600 700 800]
	fmt.Printf("Y = %v\n", Y)
	// output: Y = [300 400 500 600 700 800 900]
```

- A slice allocates new memory when an element is appended beyond its current capacity. If the capacity is less than 1024, it typically grows by a factor of 2; for larger slices, the growth factor is around 1.5. However, according to my experiment, it is not the case.

```golang
var test []int
oldCap := cap(test)
fmt.Printf("size = 0, capacity = %d\n", oldCap)

for i := 0; i < 100000000; i++ {
    test = append(test, i)
    if cap(test) != oldCap {
        fmt.Printf("size = %d, capacity = %d, ratio is %.3f\n", i+1, cap(test), 1.0*float64(cap(test))/float64(oldCap))
        oldCap = cap(test)
    }
}
```

```
size = 0, capacity = 0
size = 1, capacity = 1, ratio is +Inf
size = 2, capacity = 2, ratio is 2.000
size = 3, capacity = 4, ratio is 2.000
size = 5, capacity = 8, ratio is 2.000
size = 9, capacity = 16, ratio is 2.000
size = 17, capacity = 32, ratio is 2.000
size = 33, capacity = 64, ratio is 2.000
size = 65, capacity = 128, ratio is 2.000
size = 129, capacity = 256, ratio is 2.000
size = 257, capacity = 512, ratio is 2.000
size = 513, capacity = 848, ratio is 1.656
size = 849, capacity = 1280, ratio is 1.509
size = 1281, capacity = 1792, ratio is 1.400
size = 1793, capacity = 2560, ratio is 1.429
size = 2561, capacity = 3408, ratio is 1.331
size = 3409, capacity = 5120, ratio is 1.502
size = 5121, capacity = 7168, ratio is 1.400
size = 7169, capacity = 9216, ratio is 1.286
size = 9217, capacity = 12288, ratio is 1.333
size = 12289, capacity = 16384, ratio is 1.333
size = 16385, capacity = 21504, ratio is 1.312
size = 21505, capacity = 27648, ratio is 1.286
size = 27649, capacity = 34816, ratio is 1.259
size = 34817, capacity = 44032, ratio is 1.265
size = 44033, capacity = 55296, ratio is 1.256
size = 55297, capacity = 69632, ratio is 1.259
size = 69633, capacity = 88064, ratio is 1.265
size = 88065, capacity = 110592, ratio is 1.256
size = 110593, capacity = 139264, ratio is 1.259
size = 139265, capacity = 175104, ratio is 1.257
size = 175105, capacity = 219136, ratio is 1.251
size = 219137, capacity = 274432, ratio is 1.252
size = 274433, capacity = 344064, ratio is 1.254
size = 344065, capacity = 431104, ratio is 1.253
size = 431105, capacity = 539648, ratio is 1.252
size = 539649, capacity = 674816, ratio is 1.250
size = 674817, capacity = 843776, ratio is 1.250
size = 843777, capacity = 1055744, ratio is 1.251
size = 1055745, capacity = 1319936, ratio is 1.250
```

In C++, vector grows by 2x larger on my computer.

### array
- We can use T[...]{} to initialize an array, the size of which is deduced from the number of elements in curly brace.

- The sha256.Sum256([]byte) can generate a [32]byte array

- We can use == to compare two arrays but not slices

- When an array is used as a function input parameter, the function **deep copy** the original array. So mutation happened on the copied array doesn't go to the original input. Use *[]T to avoid copy.

## Map
- unlike in c++, we cannot take the address of a value inside a map because the map may rehash and invalidate the original pointer. This feature is really similar to the unordered_map in c++, which invalidate all previously returned iterators upon rehashing.

## Struct
- the following kind of initialization required every field to be offered a value. In other words, if some member is not exported, the following strategy cannot work.
```golang
type Point struct{ X, Y int }
p := Point{1, 2}
```

### Anonymous Fields
```golang
type Point struct {
    X, Y int
}

type Circle struct {
    Point
    Radius int
}

type Wheel struct {
    Circle
    Spokes int
}
```
We say that a Point is embedded within Circle, and a Circle is embedded within Wheel.

Anonymous fields can be used to expose functions. For instance, if we define the following two struct in package *tools*, 
```golang
type secret struct {
	Sec int
}

func (s secret) Say() {
	fmt.Printf("secret reveals itself\n")
}

type Public struct {
	secret
}
```
then we cannot create an instance in package *main*. However, we can create an instance of Public, through which the member variable Sec and function Say() are available for use.

### Marshal and Unmarshal
json.Marsh() can convert a struct into a []byte. But a member must meet all of the following requirements to be marshalled:

- it is exported, that is, begin with an upper-cased alphabet
- it does not have a json tag `json:"-"`
- it belongs to: basic types such as string, int, bool, or slices, maps, structs, pointers

Exported anonymous embedded structs are not marshaled.

## Closure pitfall's c++ counterpart
Inside a for loop, using the iteration variable in a go routine or a callback function is a well-known pitfall. I happened to create a counterpart in c++ to help understand such phenomenon:

```c++
vector<string> temp = {"0","1","2","3","4","5"};
vector<function<void()>> callbacks;
int i = 0;
for(;i<temp.size()-1;++i) {
    callbacks.emplace_back([&]() {
        cout<<temp[i]<<endl;
    });
}
for(auto f: callbacks) {
    f(); // always output 5
}
return 0;
```

## Method
### Non-pointer receiver
A method with a non-pointer receiver wouldn't change the member of the struct in that a copy is created for use inside that method.

A method with pointer receiver cannot be called on a temporary object(a rvalue in c++).

## Embedded struct
When a struct(could also be a type) A is embedded in another struct B, its exported members, including variables and methods are *promoted* to B. It is a mistake to think it's *inheritance*, as we see in Java or C++. In fact, we should think that B *has-a* A, not *is-a* A. Trying to pass a pointer or instance of B as parameters where A is required cause compilation error:

```golang
type foo struct {
	val int
}

type bar struct {
	foo
}

func Increase(f *foo) {
	f.val++
}

func main() {
	var b bar
	Increase(&b) // cannot use &b (value of type *bar) as *foo value in argument to Increase
	Increase(&b.foo) // valid
}
```

## Method Value VS Method Expression
- Method value: var_name + "." + method_name. It "remembers" the receiver, so when such value is called, we don't need to input the variable name.

- Method expression: struct_name + "." + method_name. When we call a method expression, we must send in a variable of that struct.

```golang
type bar struct {
	val int
}

func (b *bar) Add(num int) {
	b.val += num
}

func (b *bar) String() string {
	return fmt.Sprintf("%d", b.val)
}

func main() {
	var tmp bar
	method_value := tmp.Add  // var_name.method_name
	method_expression := (*bar).Add  // struct_name.method_name
	method_value(100)
	fmt.Println(tmp) // output: {100}
	method_expression(&tmp,1000)
	fmt.Println(tmp) // output: {1100}
}
```

Here is the c++ counterpart:
```c++
struct bar {
    int val;
    bar(): val(0) {}
    void Add(int num) {
        val += num;
    }
    string toString() const {
        return to_string(val);
    }
};

int main() {
    bar tmp;
    auto mimic_method_value = std::bind(&bar::Add, &tmp, std::placeholders::_1);
    auto mimic_method_expression = std::bind(&bar::Add, std::placeholders::_1, std::placeholders::_2);
    mimic_method_value(100);
    cout << tmp.toString() << endl;
    mimic_method_expression(tmp, 1000);
    cout << tmp.toString() << endl;
    return 0;
}
```

## fmt.Print
If a struct only has a String() method with pointer receiver, call fmt.Print() for an instance(not pointer) of such struct wouldn't trigger the String() unless we call in this way: fmt.Printf(&x).

In fact, adding a String() method to some struct/type makes it implement an interface called fmt.Stringer:

```golang
package fmt
type Stringer interface {
	String() string
}
```

## Forcing Inheritance
In golang, there is no key words like `extends` or `implements`, but we can use the following norm to ask compiler to check the inheritance relationship.

Assuming we have a interface named Foo, and we need to force a struct Bar to satisfy it, we can write:

```golang
var  _ Foo = (*Bar)(nil)
```

## Interface
### Comparing two interface
Two interface values `a` and `b` are comparable using == or != if:
- They hold the same dynamic type

- The underlying dynamic values are comparable

**Panic danger**:

If the dynamic value is not comparable, comparing interfaces will panic at runtime. `Maps`, `slices`, and `functions` are not comparable. Comparing them directly using `==` or `!=` can be detected by compiler, but wrapping them in interfaces and comparing will panic.

### Comparing with nil
An interface has two component under the hood:
- Type: the dynamic type of the value stored.

- Value: the actual dynamic value.

An instance is nil only when both of them are nil. In other words, it is not pointing to a variable or an instance.

```golang
type Foo struct {
	val int
}

func (f *Foo) Val() int {
	return f.val
}

type ValGetter interface {
	Val() int
} 

func main() {
	var it ValGetter 
	fmt.Println(it == nil) // true
	var foo *Foo = nil
	it = foo
	fmt.Println(it == nil) // false
}
```


