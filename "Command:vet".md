# Command - Vet
Go vet command is a great help while writing your code. It helps you detect any suspicious, abnormal, or useless code in your application. The command is actually composed of several sub analyzers and could even work with your custom analyzer.
Lets take a look at the built in analyzers first

The list of the build-in analyzers is available from the command go tool vet help. Let’s begin with the most used ones to get a better understanding.
# atomic
This analyzer will prevent any irregular usage of the atomic function
```golang
func main() {
   var a int32 = 0

   var wg sync.WaitGroup
   for i := 0; i < 500; i++ {
      wg.Add(1)
      go func() {
         a = atomic.AddInt32(&a, 1)
         wg.Done()
      }()
   }
   wg.Wait()
}
main.go:15:4: direct assignment to atomic value
```
The variable a is incremented thanks to the atomic memory primitives function addInt that is concurrent-safe. However, we assign the result to the same variable, which is a not a concurrent-safe write operation. This a careless mistake detected by the atomic analyzer.

# copylocks
A lock should never be copied, as explained in the documentation. Indeed, internally it manages the current state of the lock. As soon as the lock is used, a copy of this lock would copy its internal state, making the copy of the lock the same state as the original one rather than a new initialized one.
```golang
func main() {
   var lock sync.Mutex

   l := lock
   l.Lock()
   l.Unlock()
}
from vet: main.go:9:7: assignment copies lock value to l: sync.Mutex
```
A struct that uses a lock should be used by pointer in order to keep the internal state consistent:
```golang
type Foo struct {
   lock sync.Mutex
}

func (f Foo) Lock() {
   f.lock.Lock()
}

func main() {
   f := Foo{lock: sync.Mutex{}}
   f.Lock()
}
from vet: main.go:9:9: Lock passes lock by value: command-line-arguments.Foo contains sync.Mutex
```
# loopclosure
When you launch a new goroutine, the main one will continue to execute. The code of the goroutine and its variables will be evaluated at the execution, and it could lead to some common mistakes where a variable is used while it is still updated by the main goroutine:
```golang
func main() {
   var wg sync.WaitGroup
   for _, v := range []int{0,1,2,3} {
      wg.Add(1)
      go func() {
         print(v)
         wg.Done()
      }()
   }
   wg.Wait()
}
3333
from vet: main.go:10:12: loop variable v captured by func literal
```
# lostcancel
Creating a cancellable context from the main one will return the new context along with a function able to cancel this context. This function can be used at any time to cancel all operations linked to this context, but should always be called in order to not leak any context:
```golang
func Foo(ctx context.Context) {}

func main() {
   ctx, _ := context.WithCancel(context.Background())
   Foo(ctx)
}
from vet: main.go:8:7: the cancel function returned by context.WithCancel should be called, not discarded, to avoid a context leak
```
If you need more details about the context, their possible deviations, and the role of cancel function, I suggest you read my article about context and the cancellation by propagation.
# stdmethods
The stdmethods analyzer will make sure the methods you have implemented from the interfaces of the standard library are well compatible with what is expected:
```golang
type Foo struct {}

func (f Foo) MarshalJSON() (string, error) {
   return `{a: 0}`, nil
}

func main() {
   f := Foo{}
   j, _ := json.Marshal(f)
   println(string(j))
}
{}
from vet: main.go:7:14: method MarshalJSON() (string, error) should have signature MarshalJSON() ([]byte, error)
```
# structtag
Tags are strings in struct that should follow the convention defined in the reflect package. An extra space would make a tag not valid and could be hard to debug without vet command:
```golang
type Foo struct {
   A int `json: "foo"`
}

func main() {
   f := Foo{}
   j, _ := json.Marshal(f)
   println(string(j))
}
{"A":0}
from vet: main.go:6:2: struct field tag `json: "foo"` not compatible with reflect.StructTag.Get: bad syntax for struct tag value
```
There are many more analyzers available in vet command, but this is not where this command is powerful. It actually allows to define our own analyzers.
# Custom analyzers
If the built-in analyzers are quite useful and powerful, Go gives us the possibility to create our own analyzers.
I will use the custom analyzer I have built to detect the usage of the context package in the function arguments that you can find in the article “Build Your Own Analyzer”.
Once your custom analyzer is done, you can use it directly in the vet command:
go install github.com/blanchonvincent/ctxarg
go vet -vettool=$(which ctxarg)
You can even build your own analysis tool.
Custom analysis command
Since analyzers are totally decoupled from the commands, you can create your own command with the analyzers you would need. Let’s see an example of a custom command that uses only some analyzers we would need:

custom command based on custom list of analyzers
Building and running the command will give us a tool based on the selected analyzers:
./custom-vet help
custom-vet is a tool for static analysis of Go programs.

Registered analyzers:

    atomic       check for common mistakes using the sync/atomic package
    ctxarg       check for parameters order while receiving context as parameter
    loopclosure  check references to loop variables from within nested functions
You could also create your custom set of analyzers and merge them with the built-in analyzers you like and get a custom command that fits your own workflow and the coding standard in your company.
