# Memory Management

## Abstract 
In this session we will focus on how Go deals with memory allocations. We will dive into the stack and heap and how the compiler decides which vars goes where. We will learn how allocations on the stack vs heap affect performance (mainly GC) and what tools are available to us for optimizations.

## Design decisions
* Go is value oriented language like C-like systems languagues in oposition to managed runtime languages, where we have reference-oriented approach
* We have direct controll over the memory layout of structures
* We can collocate fields that have related values, which helps with cache locality


## Memory types
* Stack
* Heap

### Stack memory:

This is a block of memory allotted for each function to execute its defined task. Size of this memory block is decided at the compile time.

### Heap memory

This is a block of memory allotted at runtime which is demanded by our software.


### Why is this important ?

In Go , even if you use function ‘new’ to allocate memory, it might not be allocated in heap memory. It could possibly take a place in stack memory itself. **This decision will be made by Go compiler**. But why?
When you say memory is dynamically allcatted at runtime, then it has to be properly freed up once its purpose is done. Otherwise, at one point in time, our application will run out of memory resources and it could lead to a crash. Next question might come to your mind is, Go is garbage collected (GC) language. Why should we worry about it? GC would do its job of collecting the inaccessible memory and put them back to free bucket.
Still, it is important to effectively use heap memory. Reason being, when GC starts running frequently, our application routines gets a lesser share of CPU cores compared to GC. When you put pressure on GC to collect more memory from the heap, then it could possibly slow down our application performance (which may not be the case if we would have cautiously written our code).

* Sharing down the stack **tipically** does not cause heap allocations
* Sharing up the stack **tipically** does cause heap allocations

### Escape Analysis

Go compiler ```go tools compile``` takes some option and can show compiler optimizations, when we build with ```-m``` flag.

 
 
 ```go build -gcflags “-m”```

How does the compiler decides  what goes where? It checks a memory that it really needs to be allocated at a heap or it could be managed within a stack itself. The decision would be made by checking whether a particular memory needs to be shared across stack frames (function boundaries). If not, it would place it in a stack, else in heap memory.
 
**Note** 
As of go 1.14 the detailed escape analysis diagnostics `-m=2` now work again. This had been dropped from the new escape analysis implementation in the previous release. 
 
 #### Example 1
 ```
package main

import "fmt"
func main() {
    a := new(int)
    b := new(int)
    sum := *a + *b
    fmt.Println("sum = ", sum)
}
 ```

 
 and you get 
 
 ```
 ...
./escapeanalysis.go:6:10: main new(int) does not escape
./escapeanalysis.go:7:10: main new(int) does not escape
...
 ```
 
 #### Example 2
 
 ```
package main
 
import "fmt"

func main() {
 a := new(int)
 fmt.Println("value of a: ", a)
 b := new(int)
 sum := *a + *b
 fmt.Println("sum = ", sum)
}
 ```


```
...
./escapeanalysis.go:7:14: a escapes to heap
./escapeanalysis.go:6:10: new(int) escapes to heap
...
```

Why **a** escapes to the heap ?


#### When does a variable escapes to the heap
* When a value clould possibly be refrenced after the function that constructed the value returns
* When the compiler determines a value is too large to fir on the stack
* When the compiler does not know the size of the of a value at compile time


#### Commonly allocated values on the Heap
* Values shared with **Pointers**
* Values stored in **Interface** variables
* **Func literal** variables 
** Variables captured by closure
* Backing Data of **Maps, Channels, Slices, and Strings**
 

##### Which Stays on the Stack


```
package main
 
import "fmt"

func main() {
 a :=  read()
 fmt.Println("value of a: ", a)
}

func read() []byte {
 b:=make([]byte,32)
 return b
}

```

```
package main
 
import "fmt"

func main() {
 a:=make([]byte,32)
 read(a)
 fmt.Println("value of a: ", a)
}

func read(b []byte) []byte {
 // write into slice
}

```

This is why the dorcy looking interfaces like ```io.Reader```

```
type Reader interface {
 Read(p []btes) (n int, err, error)
}
```
VS
```
type Reader interface {
 Read(n int) (b []byte, err, error)
}
```

## Key takeaways
* Optimize for corectness, not performance
* Go only puts function vars pon the stack, if it can **prove** a variable is **not used** after the function returns
* Sharing down typically goes on the stack
* Sharing up typically goes to the heap
* Use the tools to figure it out

## References
* [Stack or Heap](https://golang.org/doc/faq#stack_or_heap)
* [Escape Analysis](https://medium.com/faun/golang-escape-analysis-reduce-pressure-on-gc-6bde1891d625)
