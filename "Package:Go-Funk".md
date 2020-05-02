# Go-Funk - Bridge from Go to Javascript,Python

If you’re coming from languages like Javascript or Python, you could have some surprises programming in Go.
You’ll quickly notice that all those functions such as filter, map or reduce are not part of this ecosystem. 
The fact that you don't have a built-in function to check if an element is part of a slice is even more shocking. 
When you'd simply do: ```[1, 2, 3, 4, 5].includes(5)``` in Javascript or: ```5 in [1, 2, 3, 4, 5]``` in Python, you'll discover that it's a whole other story in Go.

The first reflex, of course, is to Google the classical “golang check if element in slice” hoping to find a built-in way of doing such operations. 
The first links on the result’s page will quickly put an end to your expectation. You’ll learn that Go doesn’t have a built-in way to handle those functions.

Fortunately, there is a small library called ```Go-Funk```. Go-funk contains various helper functions such as ```Filter, Contains, IndexOf```, etc. I will cover a few of them in this article and show you how to avoid a few traps.
The issue with that kind of helpers is that it’s extremely difficult to make a single generic function that handles all the types while keeping the type safety. Let’s take the Contains function of Go-Funk for example. This function works with all the possible types. Unfortunately, it will be impossible for the compiler to check the types of the values passed as parameters. See by yourself the function's signature:

```golang

func Contains(in interface{}, elem interface{}) bool
```
To try to fix this issue, Go-Funk implements specific helper function for the standard Go types. For example, the function ```ContainsInt``` exists specifically to handle the case where we'd want to check if an int is in a slice of int. 
We'll see later that it can be trickier with custom types.

### Examples
### Map
The Map function iterates through the elements of a slice (or a map) and modifies them. **It returns a new array.**
```golang
package main

import (
	"fmt"
	"github.com/thoas/go-funk"
)

func main() {
	baseSlice := []int{1, 2, 3, 4, 5}
	newSlice := funk.Map(baseSlice, func(x int) int {
		return x + 1
	})

	fmt.Println(baseSlice)
	fmt.Println(newSlice)
}
```
Here is the result of you run this code:
```
[1 2 3 4 5] 
[2 3 4 5 6]
```
The result is what we could expect from this code. But if you have a closer look, you’ll see an issue with the type of the newSlice variable: it is interface{}. 
To fix this problem, we'll make use of type assertions. In order to do this, you'd just have to add .([]int) right after funk.Map(...). This way, we're telling the Go compiler: "You can't determine what's the type of this value, but I assure you it's a slice of int, so could you please convert it ?".
```golang
package main

import (
	"fmt"
	"github.com/thoas/go-funk"
)

func main() {
	baseSlice := []int{1, 2, 3, 4, 5}
	newSlice := funk.Map(baseSlice, func(x int) int {
		return x + 1
	}).([]int)

	fmt.Println(baseSlice)
	fmt.Println(newSlice)
}
```
The only issue with type assertion is that the errors aren’t caught during compiling but only during runtime. So you have to be extra careful when using this method.

### Filter
Filter takes a slice and a callback function as an argument. The callback function returns a Boolean. Filter passes the slice's elements one by one through the callback function: if it returns true, the element will be included in the new slice.
```golang
package main

import (
	"fmt"
	"github.com/thoas/go-funk"
)

func main() {
	baseSlice := []int{1, 2, 3, 4, 5}
	newSlice := funk.Filter(baseSlice, func(x int) bool {
		return x != 2
	}).([]int)

	fmt.Println(baseSlice)
	fmt.Println(newSlice)
}
```
In this situation, we iterate through the numbers and say: “take all of them except if the number is equal to 2”. Here is the result:
```
[1 2 3 4 5]
[1 3 4 5]
```
Note: In that situation again, we had to make use of a type assertion. This will be the case during most of the examples.

### Reduce
The Reduce function takes the following arguments: a slice, a callback function and, an initial value for the accumulator. In Go-Funk, this function returns a float. Here is how you'd use it:
```golang
package main

import (
	"fmt"
	"github.com/thoas/go-funk"
)

func main() {
	baseSlice := []int{1, 2, 3, 4, 5}
	sum := funk.Reduce(baseSlice, func(acc, elem int) int { return acc + elem }, 0)

	fmt.Println(baseSlice)
	fmt.Println(sum)
}
```
As you can see, the function iterates through the slice element by element. The callback function adds the current element to the accumulator. Result:

```
[1 2 3 4 5]
15
```
With the Reduce function, the type assertion is not necessary. If you have a look at the function signature, you'll notice that it will always return a float.
