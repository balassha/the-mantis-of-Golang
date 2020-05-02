# Go Test Package - The Unknown

go test command is probably the command people use the most in Go. However, there are some interesting details or usage you might not know about. Let’s start with the testing itself.

### Skip the test cache
In Go, if you run your tests twice in a row, you can see that is actually running only once if they all pass at the first attempt. Indeed, the tests use a cache system so they don’t re-run the tests that did not change. Let’s try with running the tests of the math package:
```
root@91bb4e4ab781:/usr/local/go/src# go test ./math/
ok     math   0.007s
root@91bb4e4ab781:/usr/local/go/src# go test ./math/
ok     math   (cached)
```
The content of the tests is not the only thing that Go is checking, it also checks the environment variables and the command-line arguments. Updating an environment variable or adding a flag will not use the cache:
```
go test ./math/ -v
[...]
=== RUN   ExampleRoundToEven
--- PASS: ExampleRoundToEven (0.00s)
PASS
ok   math 0.007s
```
Running it again will now use the cache. The cache is a hash of the content, environment variables, and command-line arguments. 
Once calculated, it is dumped to the folder $GOCACHE, that points by default on Unix to $XDG_CACHE_HOME or $HOME/.cache. 
Cleaning this folder will therefore clean the cache. Regarding the flags, as explained in the documentation, not all flags are cacheable:
The rule for a match in the cache is that the run involves the same test binary and the flags on the command line come entirely from a restricted set of ‘cacheable’ test flags, defined as -cpu, -list, -parallel, -run, -short, and -v.[…]. 
To disable test caching, use any test flag or argument other than the cacheable flags. 

The idiomatic way to disable test caching explicitly is to use -count=1. Since count defines the number of time the tests has to run, -count=1 explicitly says the tests should run exactly once, no more and no less. That makes it the perfect candidate for the idiomatic way to skip the cache.
We should note that before Go 1.12, you were able to bypass the cache thanks to the environment variable GOCACHE: GOCACHE=off go test math/.

While running the tests, Go will run them package per package. The way Go handles the package name for testing offers us more strategies for testing.

### White box testing vs black box testing
The black box testing consists of testing your code without any knowledge about the internal struct — only the exported functions and structure are available — while white box testing allows us to test the internal implementation via non-exported functions. Go natively supports both. Here is an example of a simple program where we will see the advantages of writing black and white box tests:
```golang
package deck

import (
	"errors"
	"math/rand"
)

var Empty = errors.New("Empty deck")

type Deck struct {
	cards []uint8
	shuffled bool
}

func NewDeck(numbers uint8) *Deck {
	cards := make([]uint8, 0, numbers)
	for i := uint8(0); i < numbers; i++ {
		cards = append(cards, i+1)
	}

	d := Deck{ cards: cards }

	return &d
}

func (d *Deck) Draw() (card uint8, err error) {
	if !d.shuffled {
		d.shuffle()
	}

	if len(d.cards) == 0 {
		return 0, Empty
	}
	card, d.cards = d.cards[0], d.cards[1:]

	return card, nil
}

func (d *Deck) shuffle() {
	rand.Shuffle(len(d.cards), func(i, j int) {
		d.cards[i], d.cards[j] = d.cards[j], d.cards[i]
	})
	d.shuffled = true
}
```
This code just shuffles a card deck and allows the user to draw a card. The black box tests will ensure we can create a deck and draw cards until the end:

```golang
package deck_test

import (
	"github.com/blanchonvincent/test-package/card"
	"github.com/stretchr/testify/assert"
	"testing"
)

func TestDeckCanDrawCards(t *testing.T) {
	var num uint8 = 10

	d := deck.NewDeck(num)
	for i := uint8(0); i < num; i++ {
		_, err := d.Draw()
		assert.Nil(t, err)
	}
	_, err := d.Draw()
	assert.Equal(t, err, deck.Empty)
}
```
The only condition to write a proper black box test is to add the suffix _test to the package name. It is considered as a different package and therefore does not have access to the non-exported functions. It is natively supported by Go and the compiler will not complain having two different packages in the same folder.
The white box test will ensure the deck is shuffled only once when the first card will be drawn:
```golang
package deck

import (
	"fmt"
	"github.com/stretchr/testify/assert"
	"testing"
)

func TestDeckShouldBeShuffledOnce(t *testing.T) {
	var num uint8 = 5

	d := NewDeck(num)
	assert.Equal(t, len(d.cards), int(num))
	assert.Equal(t, d.shuffled, false, "Deck should init as not shuffled")
	orderBefore := fmt.Sprint(d.cards)

	d.shuffle()
	assert.Equal(t, d.shuffled, true, "Deck has not been marked as shuffled")
	orderAfter := fmt.Sprint(d.cards)

	assert.NotEqual(t, orderBefore, orderAfter, "Deck once shuffled should have new card order")
}
```
The test just uses the same package name and can now access to the non-exported functions.
However, white box testing has one constraint. Testing your exported functions in black box testing ensures the results are valid regardless of the internal implementation of the package. We are therefore free to change and improve the internal implementation without breaking any tests. With white box testing, since the tests are tied to the internal implementation, improving it could break the tests.
Now, let’s move to the other feature of this test package, the benchmarks.

### Run your benchmarks once with your tests
Introduce in Go 1.12, -benchtime=1x, -benchtime=10x, etc. allows you to run your benchmark only the number of times you would like. The flag -benchtime=1x can be useful with your tests suite since it would allow you to run your benchmarks at least once in order to check that they are not broken due to your last change.
Before Go 1.12, we were able to achieve the same result with the flag -benchtime=1ns that stops the loop in the benchmarks after 1ns. Since 1ns is the smallest unit of time, it would run your benchmarks only once. Benchmarks allow you to get metrics like time to perform an operation, memory used, or the number of allocations made on the heap. Since Go 1.13, you can report much more than that.

### Report your custom metrics
Go 1.13 comes with a new method reportMetric that allows us to report our custom metrics. Let’s use the previous example with the deck and modify the code to shuffle the cards between 1 and 20 times before drawing the first one. Here is an example with reporting the number of shuffles with the benchmark:
```golang
package deck

import (
	"testing"
)

func BenchmarkGC(b *testing.B) {
	b.ReportAllocs()
	shuffled := 0

	for i := 0; i < b.N; i++ {
		d := NewDeck(100)
		_, _ = d.Draw()
		shuffled += int(d.shuffled)
	}

	b.ReportMetric(float64(shuffled)/float64(b.N), "shuffle/op")
}
```
And the generated result:
```BenchmarkDeckWithRandomShuffle-8       88666        12389 ns/op            5.15 shuffle/op        144 B/op         2 allocs/op
PASS
ok     1.529s```
As we can see in the result, Go 1.13 brought another change. It does not round B.N anymore and use accurate number. This CL allows the package to reduce the impact of external noise such as GC on benchmarks, especially for the benchmarks with a very small wall time. The benchmark also runs faster:
```// go 1.13
BenchmarkDeckWithRandomShuffle-8      88666         12389 ns/op
PASS
ok     1.529s
// go 1.12
BenchmarkDeckWithRandomShuffle-8      100000        12765 ns/op
PASS
ok     1.890s```
