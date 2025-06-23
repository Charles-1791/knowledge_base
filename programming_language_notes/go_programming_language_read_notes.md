# Go Programming Language Read Notes

## compilation
- "go run xxx.go" compiles and execute a single go file(with `main()` function).

- "go build xxx.go" creates an executable binary in the current directory but not executes it

- The function `init()` is a special initialization function in Go. It is automatically invoked by the runtime before the `main()` function runs, and you cannot call `init()` explicitly in your code.

## syntax
- a++ is a statement, not a expression. statement like `b = a++` is invalid.

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
- `strings` are immutable. To modify a character within a `string`, you need to convert the `string` into a `[]rune`, make the modification, and then convert it back to a `string`.

- Go strings follow the `UTF-8` standard, characters (`runes`) may be composed of multiple bytes. Therefore, the correct way to iterate over characters in a string is to convert it to a []rune and iterate over the runes.

- Alternatively, a for loop over a string like `for _, r := range str` iterates rune by rune, not byte by byte, decoding `UTF-8` correctly.

- Converting a string into a []rune allocates new memory. Since rune is fixed-lengthed, the size of the converted []rune is at least as large as the original string's byte length

unsafe.Sizeof(x) returns a string's head size, which is 16 on a 64-bit system. Internally, a string in golang having the following structure:
```golang
type string struct {
    Data *byte  // pointer to the actual UTF-8 bytes
    Len  int    // length in bytes
}
```

Converting a string to []byte also necessitates memory allocation, and so does the reverse.

## Array and Slice
### Slice
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
Using the subscript operator `X[i:j]` to create a sub-slice `Y` from a slice `X` does not allocate new memory. Instead, `Y` shares the same underlying array as `X`.

This means that mutating elements of `Y` will also affect `X`, since they both refer to the same data.

However, if you append to `Y` and the capacity of `Y` is exceeded, a new array is allocated and the existing elements are copied over. From that point on, `Y` refers to a different array, and further mutations no longer affect `X`.

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

A slice allocates new memory when an element is appended beyond its current capacity. If the capacity is less than 1024, it typically grows by a factor of 2; for larger slices, the growth factor is around 1.5. However, according to my experiment, it is not the case.

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

In comparison, the `vector` in C++ grows by 2x larger on my PC (x64, MacOS).

### array
- We can use T[...]{} to initialize an array, the size of which is deduced from the number of elements in curly brace.

- The sha256.Sum256([]byte) can generate a [32]byte array

- We can use == to compare two arrays but not slices

- When an array is used as a function input parameter, the function **deep copy** the original array. So mutation to a copied array doesn't affect the original one. Use *[length]T instead if you intend to change the elements or avoid copy.

## Map
Unlike in c++, we cannot take the address of a value inside a map because the map may rehash and invalidate the original pointer. This feature resembles the behavior of `unordered_map::iterator` in c++, which invalidate all previously returned iterators upon rehashing.

## Struct
The following form of struct initialization requires all fields to be provided in order:
```golang
type Point struct{ X, Y int }
p := Point{1, 2}
```
This approach cannot be used if any field is unexported (i.e., starts with a lowercase letter), because unexported fields cannot be accessed outside the defining package.

### Marshal
json.Marshal() can convert a struct into a []byte, typically containing a JSON representation. However, a struct field must meet all of the following conditions to be marshaled:

- It must be exported, that is, begin with an uppercase letter.
- It does not have a json tag `json:"-"`.
- Its type must be supported — basic types like string, int, bool, as well as slices, maps, structs, and pointers to those types.

Additionally:
- Exported anonymous embedded structs are marshaled, but only their exported fields are included.
- If an embedded struct is unexported, it is ignored.

An unexported struct type in Go can be marshaled by json.Marshal only if the call occurs within the same package where the type is defined. In that case, reflection can access the exported fields and include them in the output. However, if the type is unexported and used from another package, its structure is hidden to reflection, and marshaling it will result in an empty JSON object ({}).

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

### Costs of Using an Interface
An interface in golang is two-word long, which is 16 bytes on a 64-bit machine. The first word stores a pointer to an internal structure containing the concrete type information and a method table. And the second pointer points to the address of the concrete instance.

A method table in Go is similar to a virtual table (vtable) in C++ — it acts as a jump table that stores function pointers for each method declared in the interface. As a result, calling a method on an interface incurs more overhead than calling a concrete method directly. It involves dereferencing the interface's first word to access the `itab`, then using a fixed offset to fetch the appropriate function pointer from the method table, and finally performing an indirect call with the data pointer as the receiver.

```golang
interface {
    itab *itab     // pointer to interface-table
    data *T        // pointer to concrete value
}
```


## Switch Clause
In golang, a switch clause is like a multiple if-elseif-else: all the cases are evaluated in order, whereas in c++, where the switch key must be an integer or enum, compiler may build jump table or a binary search tree to achieve faster matching.

The evaluate-cases-in-order feature demands extra attention to a type switch, where the key is an interface. 

## A Trick -- "Retract" What You Printed in A Bash
- Check how many characters have been output, say N. 
- Generate a string containing N blank spaces with `strings.Repeat(" ", N)`
- Return the cursor to the start of the current line, print the blank spaces, like `fmt.Printf("/r%s", blankspaces)`

We can use this trick to print "LOADING...":

```golang
func printWaiting(stopChan <-chan struct{}) {
	strs := "LOADING..."
	padding := strings.Repeat(" ", len(strs))
outer: for {
		select {
		case <- stopChan:
			break outer
		default:
			fmt.Print("\r")
			for i:=0; i<len(strs); i++{
				fmt.Printf("%c",strs[i])
				time.Sleep(time.Millisecond * 200)
			}
			fmt.Print("\r" + padding)
		}
	}
}
```

## Blank Import
Sometimes we see imports like this:
```golang
_ "github.com/go-sql-driver/mysql"
```
The purpose is to trigger an init() function defined inside that package, which usually initializes some variables or setup certain entries. For instance, in the `driver.go` of the `mysql` package, an entry is inserted into a global variable in `sql` package:
```golang
func init() {
	if driverName != "" {
		sql.Register(driverName, &MySQLDriver{})
	}
}
```
When you call sql.Open to create a connection to MySQL, the stored driver is retrieved to do the real job.
```golang
func Open(driverName, dataSourceName string) (*DB, error) {
	driversMu.RLock()
	driveri, ok := drivers[driverName]
	driversMu.RUnlock()
	if !ok {
		return nil, fmt.Errorf("sql: unknown driver %q (forgotten import?)", driverName)
	}

	if driverCtx, ok := driveri.(driver.DriverContext); ok {
		connector, err := driverCtx.OpenConnector(dataSourceName)
		if err != nil {
			return nil, err
		}
		return OpenDB(connector), nil
	}

	return OpenDB(dsnConnector{dsn: dataSourceName, driver: driveri}), nil
}

```

## GO.MOD & GOCACHE & GOMODCACHE
After golang 1.11, a project does not need to locate inside `$GOPATH`. By running `go mod init your_module_name`, a `go.mod` and a `go.sum` file are generated.

When you build your module, before linking happens, each packages was compiled into a `.a` file, which is stored in `GOCACHE`. You can check your path with:
```bash
go env GOCACHE
```

`GOMODCACHE` is the path where dependency source codes are downloaded when `go get`(or `go build`, which triggers `go get`) is called.

```bash
go env GOMODCACHE
```

Both GOCACHE and GOMODCACHE can be modified like this:
in ~/.bashrc
```bash
export GOMODCACHE=your_path
export GOCACHE=your_path
```

## Go External Test
Suppose package `B` imports package `A`, and we want to add some `_test.go` files to package `A`. However, these tests need to use variables or functions defined in `B`. To avoid cyclic imports, we place the tests in an external test package named `A_test`. This external test package is not located in a separate directory; instead, all `_test.go` files reside alongside the production code files within package `A`’s directory.

Since `A_test` is a separate package, its test files cannot access unexported members of package `A` directly. To work around this, we create a special Go file inside package `A` (usually named `export_test.go`) that selectively exposes internals only for the tests. 

A real `export_test.go` looks like:

```golang
package fmt

var IsSpace = isSpace
var Parsenum = parsenum
```

## Reflect
### Value.Elem() VS Type.Elem()
Both `reflect.Value` and `reflect.Type` have a member function `Elem()`, but they have distinct meanings.
| caller | return value | meaning| valid caller type |
|-|-|-|-|
|reflect.Value|reflect.Value|"dereferencing" the address where pointer or interface points to| Interface, Pointer|
| reflect.Type|reflect.Type| what is the type of the elements inside the container | Map, Slice, Array, Chan |

Interestingly, for a reflect.Value `x` of kind `Pointer`, for instance, `x := reflect.ValueOf(&y)`:
- `x.Elem()` returns the "reference" or "left-value"(c++) of variable y, and it is `Addressable`. It is also settable, that is,`CanSet() == true`, if it points to an exported member.
- `x.Type().Elem()` returns the type which x points to, namely, `y`. 

### reflect.Value.Method(i) VS reflect.Type.Method(i)
Both `reflect.Value` and `reflect.Type` have a method called `Method(i)`, and they have certain disparities.

Say that we have `x := reflect.Value(&Foo{})`

#### reflect.Value.Method(i)
`x.Method(i)` returns a `method value`, namely, an `address` or a `function pointer`. It stores no information about the method name. So there is not `Name` member of `Method(i).Type()`. Recall that [method value](#method-value-vs-method-expression) stores the address of the instance into it, therefore, the first input variable is not a pointer to the caller instance. In addition, since `x.Method(i)` returns an `address`, it is callable.

#### Reflect.Type.Method(i)
x.Type() returns the metadata of a value, which carries no information about the `callable address`, as a result, `Type().Method(i)` is not callable. However, in the metadata store the method names, as well as their input parameters and return values. In contrast, `Type().Method().In(0)` is always the receiver itself.

```golang
type Phone struct {
	Owner string
}

func (ph *Phone) Call(callAt time.Duration) {
	fmt.Printf("%s made at phone call at %v", ph.Owner, callAt)
}

func TestMethodType(t *testing.T) {
	ph := &Phone{
		Owner: "Charles",
	}
	rval := reflect.ValueOf(ph)
	// Output: rval.Method(0).Type() = func(time.Duration)
	fmt.Printf("rval.Method(0).Type() = %s\n", rval.Method(0).Type().String())
	tval := reflect.TypeOf(ph) 
	// Output: tval.Method(0).Type = func(*myformatter.Phone, time.Duration)
	fmt.Printf("tval.Method(0).Type = %s\n", tval.Method(0).Type.String()) 
}

```

#### Summary
|  | Return Type | Callable? | Method name obtainable? | First parameter |
|-|-|-|-| -|
| reflect.Value.Method(i) | reflect.Value | YES | NO | First Parameter in Signature |
| reflect.Type.Method(i) | reflect.Method | NO | YES | Method Receiver |

## Unsafe
### Converting unsafe.Pointer to uintptr is hazardous
#### GC may move stack allocated object

Go uses a `non-moving` garbage collector for heap-allocated objects, which guarantees that heap pointers remain stable during GC. However, this guarantee does not apply to stack-allocated variables.

Go's stacks are small at first (typically 2 KB) and can grow. When this happens, variables on the stack may be moved to a new memory region. Normally, Go tracks and adjusts all legitimate pointers — including unsafe.Pointer — so they remain valid after a stack move.

But once a uintptr is used to store an address (converted from unsafe.Pointer), the GC no longer tracks it as a pointer. From the GC’s perspective, it's just a number. If the original object is moved (e.g., due to stack growth), the value stored in uintptr becomes stale and invalid, potentially leading to undefined behavior.

#### GC may recycle the heap or stack allocated object

Another risk arises when you convert an unsafe.Pointer to a uintptr and the original pointer is no longer referenced. Because uintptr is not considered a pointer, the garbage collector may reclaim the memory that the original pointer referred to — even though you still have the address stored in uintptr.

As a result, dereferencing that address can lead to accessing freed or repurposed memory — a classic use-after-free scenario.

