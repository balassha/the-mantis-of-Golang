# Go : Unsafe 
##### The name of the package could let us know that we should not use it. 

To understand the reasons that it could be unsafe to use this package, let’s first refer to the documentation:
Package unsafe contains operations that step around the type safety of Go programs.
Packages that import unsafe may be non-portable and are not protected by the Go 1 compatibility guidelines.
The name is therefore used as opposition of the safeness that provide the types with Go. 

Let’s dive now into those two points mentioned by the documentation.

### Type safety:- 
In Go, each variable has a type that can be converted to another type before being assigned to another variable.
During this conversion, Go performs a transformation of this data in order to fit with the requested type. 
Here is an example:
```golang
var i int8 = -1 // -1 binary representation: 11111111
var j = int16(i) // -1 binary representation: 11111111 11111111
println(i, j) // -1 -1
```

The unsafe package will give us a direct access to the memory of this variable with the raw binary value stored at this address. 
It is up to us to use it as we want when bypassing the type constraint:
```golang
var k uint8 = *(*uint8)(unsafe.Pointer(&i))
println(k) // 255 is the uint8 value for the binary 11111111
```
The raw value is now interpreted as an uint8 without using the previous declared type.

### Go 1 compatibility guidelines:- 
The guidelines of Go 1 clearly explain that the usage of unsafe package could potentially break your code after if they change the implementation:
Packages that import unsafe may depend on internal properties of the Go implementation. We reserve the right to make changes to the implementation that may break such programs.
We should keep in mind that, in Go 1, the internal implementation could change and we could face issues, where the behavior has slightly changed between two versions. 

However, the Go standard library also use the unsafe package in many places.

### Usage in Go with the reflection package
The reflection package is one of the packages that uses it the most. The reflection is based on the internal data that the empty interface contains.
To read the data, Go just converts our variable to an empty interface and reads them by mapping a struct that matches the internal representation of the empty interface with the memory at the pointer address:
```golang
func ValueOf(i interface{}) Value {
   [...]
   return unpackEface(i)
}
// unpackEface converts the empty interface i to a Value.
func unpackEface(i interface{}) Value {
   e := (*emptyInterface)(unsafe.Pointer(&i))
   [...]
}
```
The variable e now contains all information about the value, such as the type or if the value has been exported. 
The reflection also uses the unsafe package to modify the value of the reflected variable by updating the value directly in memory, as we have seen previously.

### Usage in Go with the sync package
Another interesting usage of the unsafe package is in the sync package. 
The pools are shared across all goroutines/processors via a segment of memory that all goroutines can access via the unsafe package:
```golang
func indexLocal(l unsafe.Pointer, i int) *poolLocal {
   lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
   return (*poolLocal)(lp)
}
```
The variable lis the memory segment and i is the processor number. 
The function indexLocal just read this memory segment — that contains X (number of processors)poolLocal structs — with an offset related to the index it has to read. 
Storing a single pointer to the full memory segment is a very light way to implement the shared pools.

### Usage in Go with the runtime package
Go also uses the unsafe package a lot in the runtime since it has to deal with memory operations, like stack allocation or freeing stack memory. 
The stack is represented by two boundaries in its struct:
```golang
type stack struct {
   lo uintptr
   hi uintptr
}
```
Then the unsafe package will help to make the operation:
```golang
func stackfree(stk stack) {
   [...]
   v := unsafe.Pointer(stk.lo)
   n := stk.hi - stk.lo
   // then memory freeing based on the pointer to the stack
   [...]
}
```
Also, we can also use this package in our application in some cases, like the conversion between structs.

### Usage as a developer
A nice usage of the unsafe package would be the conversion of two different structs with the same underlying data, which is impossible to achieve with the converters:
```golang
type A struct {
   A int8
   B string
   C float32

}

type B struct {
   D int8
   E string
   F float32

}

func main() {
   a := A{A: 1, B: `foo`, C: 1.23}
   //b := B(a) cannot convert a (type A) to type B
   b := *(*B)(unsafe.Pointer(&a))

   println(b.D, b.E, b.F) // 1 foo 1.23
}
```

Another nice usage made from the unsafe package is http://golang-sizeof.tips that help you to understand the size padding of your structs.
##### In conclusion, the package is quite interesting and powerful and it should be used carefully.
