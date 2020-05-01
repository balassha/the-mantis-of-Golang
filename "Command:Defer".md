# Command - Defer
Deferred function is declared inside another function and is called when surrounding function exits — either normally or if goroutine panics.
This mechanism is used e.g. to do cleanup and close open files or sockets.
Deferred functions are evaluated in LIFO fashion .

E.g.
```golang
func f() {
    fmt.Println("f()")
}
func g() {
    fmt.Println("g()")
}
func h() {
    fmt.Println("h()")
}
func main() {
    defer f()
    defer g()
    defer h()
}
```
Output:
```golang
h()
g()
f()
```

defer statement can be used anywhere inside the function so it’s not limited only to its beginning:
```golang
func f() {
    fmt.Println("boo")
}
func main() {
    fmt.Println("one")
    
    defer f()
    fmt.Println("two")
}
```
Output:
```golang
one
two
boo
```
### Anonymous functions
It’s allowed to used both, named and anonymous functions with defer statement 

```golang
func main() {
     defer func() {
         fmt.Println("foo")
     }()
}
```
Output: 
```golang
foo
```
### Evaluating deferred function
When deferred function is nil then runtime error is thrown but it’s done when outer function exists not at the moment statement is executed
```golang
func main() {
    var f func()
    defer f()
    fmt.Println("foo")
}
```
Output:
```golang
foo
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0xffffffff addr=0x0 pc=0x8d034]
goroutine 1 [running]:
main.main()
	/tmp/sandbox581176742/prog.go:11 +0xdb
```  
(11th line is the closing bracket of main function)

Function used by defer statement is evaluated at the moment statement is encountered, not when surrounding function exists
```golang
func main() {
    var f func()
    defer f()
    f = func() {
        fmt.Println("foo")
    }
}
```
Output:
```golang
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0xffffffff addr=0x0 pc=0x8ce94]

goroutine 1 [running]:
main.main()
	/tmp/sandbox806328181/prog.go:13 +0x7b
```  
  
### Evaluating parameters of deferred function
While processing defer statement arguments are evaluated. Those arguments will be used when outer function exists (source code):
```golang
func g() int {
    fmt.Println("g()")
    return 0
}
func f(int) {
    fmt.Println("f()")
}
func main() {
    defer f(g())
    fmt.Println("Hello world!")
}
```
Output:

```golang
g()
Hello world!
f()
```
If array is passed and modified before outer function exists then original (at the moment of defer statement) value will be used (source code):
```golang
func f(nums [4]int) {
    fmt.Printf("%v\n", nums)
}
func main() {
    nums := [...]int{0,1,2,3}
    defer f(nums)
    nums[0] += 1
}
```
```golang
Output is [0 1 2 3].
```
If we’ll use type which is a descriptor like slice then modified arguments will be used (source code):
```golang
func f(nums []int) {
    fmt.Printf("%v\n", nums)
}
func main() {
    nums := []int{0,1,2,3}
    defer f(nums)
    nums[0] += 1
}

Output is [1 1 2 3].
```
The same is true for e.g. maps (source code):
```golang
func f(nums map[string]int) {
    fmt.Printf("%v\n", nums)
}
func main() {
    nums := map[string]int{"one": 1}
    defer f(nums)
    nums["one"] += 1
}

Output is map[one:2].
```
### Changing return value(s) of surrounding function
If outer function has named result value(s) then deferred function has access to it through closure and can modify it (source code):
```golang
func f() (v int) {
    defer func() {
        v *= 2
    }()
    v = 2
    return
}

func main() {
    fmt.Println(f())
}

Output is 4.
```

### Return value(s) of deferred function
Whatever deferred function returns is discarded (source code):
```golang
func f() int {
    defer func() int {
        return 5
    }()
    return 0
}
 
func main() {
    fmt.Println(f())
}
Output is 0.
```
