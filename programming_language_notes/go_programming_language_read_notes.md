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
- channel

### interface types
- interface

## string
### string, []byte and []rune
```golang
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```

- `strings` are immutable. To modify a character within a `string`, you need to convert the `string` into a `[]rune`, make the modification, and then convert it back to a `string`.

- Go strings follow the `UTF-8` standard, characters (`runes`, alias of `int32`) may be composed of multiple bytes. Therefore, the correct way to iterate over characters in a string is to convert it to a []rune and iterate over the runes.

- Alternatively, a for loop over a string like `for _, r := range str` iterates rune by rune, not byte by byte, decoding `UTF-8` correctly.

- Converting a string into a []rune allocates new memory. Since rune is fixed-lengthed, the size of the converted []rune is at least as large as the original string's byte length

unsafe.Sizeof(x) returns a string's head size, which is 16 on a 64-bit system. 

Converting a string to []byte also necessitates memory allocation, and so does the reverse.

### Cost of String Comparison
Recall that a string is effective a struct of an unsafe.Pointer to an array and a `len` field, if two strings shares the same unsafe.Pointer, then the runtime only needs to check the `len` to tell whether they are equal. The time complexity is O(1). However, if their underlying pointer are different, the comparison takes O(n) time.

## Array and Slice
### Slice
- unsafe.Sizeof() returns a slice's head size, which is 24 on a 64-bit system. The following is source code of the slice:
```golang
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
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

A slice allocates new memory when an element is appended beyond its current capacity. If the capacity is no greater than 1024, it typically grows by a factor of 2; for larger slices, the growth factor gradually decreases to 1.25. The following code is the source code computing the next capacity, minor adjustments are made to the result though.

```golang
// nextslicecap computes the next appropriate slice length.
func nextslicecap(newLen, oldCap int) int {
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		return newLen
	}

	const threshold = 256
	if oldCap < threshold {
		return doublecap
	}
	for {
		// Transition from growing 2x for small slices
		// to growing 1.25x for large slices. This formula
		// gives a smooth-ish transition between the two.
		newcap += (newcap + 3*threshold) >> 2

		// We need to check `newcap >= newLen` and whether `newcap` overflowed.
		// newLen is guaranteed to be larger than zero, hence
		// when newcap overflows then `uint(newcap) > uint(newLen)`.
		// This allows to check for both with the same comparison.
		if uint(newcap) >= uint(newLen) {
			break
		}
	}

	// Set newcap to the requested cap when
	// the newcap calculation overflowed.
	if newcap <= 0 {
		return newLen
	}
	return newcap
}
```

In comparison, the `vector` in C++ grows by 2x larger on my PC (x64, MacOS).

### array
- We can use T[...]{} to initialize an array, the size of which is deduced from the number of elements in curly brace.

- The sha256.Sum256([]byte) can generate a [32]byte array

- We can use == to compare two arrays but not slices

- When an array is used as a function input parameter or for-ranged, the function **deep copy** the original array. So mutation to a copied array doesn't affect the original one. Use *[length]T instead if you intend to change the elements or avoid copy.

## Map
Unlike in c++, we cannot take the address of a value inside a map because the map may rehash and invalidate the original pointer. This feature resembles the behavior of `unordered_map::iterator` in c++, which invalidate all previously returned iterators upon rehashing.

Here is the source code:
```golang
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
```

## Slice VS Map pointer-property
Conclusion, `Map` is in fact a pointer but slice is a struct containing a pointer member.

Image there is a function taking in a slice, say `slc`, appending one element to it. After the function returns, the result of `len(slc)` is not incremented by 1. In contrast, if a function takes in a `map`, after inserting one element, the `len` returns the incremented value.

```golang
func appendToSlice(slc []int) {
	slc = append(slc, 1)
}

func insertToMap(mp map[int]int) {
	mp[1] = 1
}

func main() {
    slc := make([]int, 0)
	mp := make(map[int]int, 0)
	fmt.Printf("before append, len(slc) = %d\n", len(slc))
	fmt.Printf("before append, len(mp) = %d\n", len(mp))
	fmt.Printf("--------- insert to slice and map ---------\n")
	appendToSlice(slc)
	insertToMap(mp)
	fmt.Printf("after append, len(slc) = %d\n", len(slc))
	fmt.Printf("after append, len(mp) = %d\n", len(mp))
}
```
The execute result is:
```text
before append, len(slc) = 0
before append, len(mp) = 0
--------- insert to slice and map ---------
after append, len(slc) = 0
after append, len(mp) = 1
```
What causes such difference?

Recall the internal representation of slice is:
```golang
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
When we create a slice, go runtime internally invokes this function:
```golang
func makeslice(et *_type, len, cap int) unsafe.Pointer 
```
which returns the starting address of the array. Then, the runtime builds the `slice` struct, fill in the len, cap and returns it. In short, what you actually gets is a struct. 

When such struct is passed by value, all three members are copied into a new struct(as a function argument). Since `array` is itself an `unsafe.Pointer`, modification to its elements is visible outside the function. However, when appending an element into the slice, go runtime only modifies the `len` (and `cap`, if necessary) of the copied struct, leaving the original struct intact. As a result, when `len()` is called outside the function, the unchanged len is returned.

In comparison, When we initialize a map, go runtime internally invokes:
```golang
func makemap(t *maptype, hint int, h *hmap) *hmap
```
So what you receive is a pointer to `hmap`. Modification to the map is achieved through pointer dereference, so the size field(in source code, the real name is "count") is changed.



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

## [Before Go 1.22] Closure pitfall's c++ counterpart
BEFORE GO 1.22, inside a for loop, using the iteration variable in a go routine or a callback function is a well-known pitfall. GO 1.22 changed the semantics of iterator. I happened to create a counterpart in c++ to help understand such phenomenon:

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
For two interface values `a` and `b`, a == b iff:
- They hold the same dynamic type

- The underlying dynamic values are comparable and equal to each other

**Panic danger**:

If the underlying dynamic value is not comparable, comparing two interfaces results in panic at runtime. To be more specific, `Maps`, `slices`, and `functions` are not comparable - doing so can be detected by compiler. However, by wrapping them in interfaces, comparison can bypass compiler's check, resulting in panic.

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
An interface in golang is two-word long, which is 16 bytes on a 64-bit machine. For a non-empty interface, that is, interface containing at least one function, the first word is a pointer to an internal structure, `ITab`, which contains the concrete type information and a method table. For an empty interface, the first word stores the dynamic type info.

Regardless of its emptiness, the second pointer is always the address of the concrete instance.

```golang
type itab = abi.ITab
/// ...
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

The struct `ITab` has a method table member called `Fun`. A method table is similar to a virtual table (vtable) in C++ — it acts as a jump table that stores function pointers for each method declared in the interface. As a result, calling a method on an interface incurs more overhead than calling a concrete method directly. It involves dereferencing the interface's first word to access the `itab`, then using a fixed offset to fetch the appropriate function pointer from the method table `Fun`, and finally performing an indirect call with the data pointer as the receiver.

```golang
// The first word of every non-empty interface type contains an *ITab.
// It records the underlying concrete type (Type), the interface type it
// is implementing (Inter), and some ancillary information.
//
// allocated in non-garbage-collected memory
type ITab struct {
	Inter *InterfaceType
	Type  *Type
	Hash  uint32     // copy of Type.Hash. Used for type switches.
	Fun   [1]uintptr // variable sized. fun[0]==0 means Type does not implement Inter.
}
```
You may have noticed that the first pointer in ITab points to a interface type, and the second points to the dynamic concrete type. As a result, the ITab is shared for all {InterfaceType, ConcreteType} pair, which is not like in c++, where all instances of the same concrete class share the same vtable.

## Switch Clause
In golang, a switch clause is like a multiple if-elseif-else: all the cases are evaluated in order, whereas in c++, where the switch key must be an integer or enum, compiler may build jump table or a binary search tree to achieve faster matching. The only exception is the position of "default", which can be put before or after any case clause while maintain the identical semantics. For instance, the following switch clauses are equivalent:

```golang
switch x {
case 0:
	// ...
case 1:
    // ...
case 2:
	// ...
default:
	// ...
}

switch x {
case 0:
	// ...
default:
	// ...
case 1:
    // ...
case 2:
	// ...
}
```

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
import _ "github.com/go-sql-driver/mysql"
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

