# Garbage Collection


## Abstract

In this session we will focus on how the GC works in go. What we need to know of it and how it impacts perofrmance. We will discuss some options for GC tunning and practices, which we can use to reduce the impact on performance. We will also try to note the evolution of GC in Go where posible.


## Setting the stage
* Go programs frequently have 100K + stacks/goroutines
* goroutines/stacks are preempted at GC safepoints by the Go scheduler
* Go sheduler multiplexes  Go routines onto OS thread, count(Running OS Thread) = count (CPU Cores)*
* Go manages stacks by copying them and updating pointers. Growing Stacks are employed oposed to old Split Stacks

### Memory Management
* Go is value oriented language like C-like systems languagues in oposition to managed runtime languages, where we have reference-oriented approach
* We have direct controll over the memory layout of structures
* We can collocate fields that have related values, which helps with cache locality
* Fast FFI with C and C++ (a design choice made by google for interoperability with C family)
* Go allows interior pointers (has strong impact on GC, can keep entire struct live)
* No JIT, so no way of dynamic feedback optimizations


### GC knobs

* SetGCPercent - controlls how much CPU/Memory we want to use. Default value is 100 and means 50/50 split between live data and allocations 
* SetMaxHeap - has been under discussion/internal evaluation for alot of time and is supposed to let us defines the maximum heap size. **Not Available!**

Out of memory are issue in Go; spikes hould be handled by increaased CPU costs instead of aborting. GC should be able to detect memory pressure and allow the app to shed the load, by informing it about the situation. Also provides a lot of flexibility in scheduling. It knows how much heap it can use and can size properly.
 
## GC History

### Latency

* Latency is threat! 
* latency is comulative
* Service Level Objective (SLO) 99% of the GC cycle takes < 10ms
  * this leads to poor experiance across user sessions (only 37% get < 10ms responces in certain scenarios)
* SLO needs to get 99.99% to get 99% responces < 10 ms (Math in Google)
* Redundancy can workaround the issue, but has very high cost -> more metal, cooling, power, maintanance, space and etc, 

### 2014 GC SLO

* 25% of the total CPU
* heap = 2x live heap
* 10 ms Stop The World (SWT) GC @every 50ms
* Go routines allocations and GC assist
* **Go runtime and complier still written in C** as a side note

### 2018 GC SLO

* 25% of the CPU during GC cycle
* heap = 2x life heap or max heap*
* two < 500 μs STW pauses per GC
* Goroutines allocation + GC assists
* Minimal GC assists in steady  state

## Go's GC
* Concurent mark and sweep GC
  * mark phase - traverse all referencable objects and mark them as still in use. These objects are known as **live memory**
  * sweep phase - everything not marked is considered a garbage and needs to be cleaned up
  * does not STW for the entire GC cycle
  * mostly runs un paralel with App code

### Glocary
* heap size - peace of memmory for allocation of objects.
* Live memory - all allocations, which are curently being referenced by the application

### Marking

Marking involves traversing all the objects the application is currently pointing to, so the time is proportional to the amount of live memory in the system, regardless of the total size of the heap. In other words, having extra garbage on the heap will not increase mark time, and therefore will not significantly increase the compute time of a GC cycle.

Less frequent GC'ing = less CPU spent in marking = high memory usage

**Pacer**
Go's GC uses a pacer to determine when to trigger the next GC cycle
* modeled as a control problem -> trying to hit a target heap size goal
* a GC cycle every time the heap doubles. Sets this at the end of the mark phase. Remeber heap = 2x live memory
* GOGC used by the runtime to calculate trigger ration (I think it has been reaplced by now)

**GC assists**
* as we don't stop the world any more during GC, memory allocations need to prevent unbound growth of memory allocations
* GC assits puts the burden of memmory allocation to the goroutine, which is responsible for the allocation during GC
* GO has a dedicated GC worker hence the term **assist**
* it assists specificaly the mark phase

```
someObject :=  make([]int, 5)
```
* makes a call to ```runtime.makeslice``` which end up calling ```runtimemallocgc```

* an concept of allocation debt is introduced to manage the **fairness** of allocation between goroutines. The more you allocate, the more you need to sweep
* eventually the allocation will hit the ```gcAssistAlloc```. Which has the following algorithm
  * check if the go routine is not doing something non preemptible
  * perform GC mark work
  * if you there is no allocation debt, then return
  * goto step 2
  
* any goroutine doing allocation during GC will pay a price and will show up as latency
* more GC cycles means more GC assist -> API latency increases
* can be seen in a execution Trace during GC. The profile shows it all. All marked with "runtime.gcAssistAlloc" are doing the MARK ASSIST
  
  
## Optimization approaches

There are some optimization approaches and all of the revolve around the attempt to **free** your application of the burden of GC or to reduce it as much as possible. So far 2 ortogonal approaches are available
* influencing the GC and anticipating its effect
* design your applciation to not leave allocations that need to be GC-ted

### The Memory Balast

An approach devised to prevent the Pacer of scheduling GC too often. The idea behind is to habe a sort of a balast - a big chunk of allocated memory, which is actually never used, but during sweep will be marked as accessible and counted towards the live memory.

```
func main() {

	// Create a large heap allocation of 10 GiB
	ballast := make([]byte, 10<<30)

	// Application execution continues
	// ...
}
```
* next GC will be when the heap reaches 20 GB
* balast is considered live memory
* if you have many short lived allocations, heap will drop significantly in next GC cycle
* You can impact the peacer by picking a size of the balast, which delays the GC based on your needs
* as long as you don't touch the balast it remains in the Virtaul Memory and does not lead to allocation in the Resident Set Size -> phisical allocations
* improved latency as we have less GC cycles and less GC assits e.g. more CPU for the app

### sync.Pools

Allows you to reduce the memory allocations by giving you a pool of reusable objects

```
The comments on sync/pool.go  say that:
A Pool is a set of temporary objects that may be individually saved and retrieved.
A Pool is safe for use by multiple goroutines simultaneously.
```

* very handy in http servers, which nneds to marshal streams into json structures
* pool type is to reuse memory between garbage collections
* it should not circumvent garbage collection, it should only make it more efficient

**Usage**

* construct it

```
var bufferPool = sync.Pool{
   New: func() interface{} {
      return new(bytes.Buffer)
   },
}
```

* Now you have pool which will be create and new buffers. You can get you first buffer here:
```
buffer := bufferPool.Get().(*bytes.Buffer)
```
* Method get will return already existing *bytes.Buffer in the pool and if it’s not will call method New which init new ```*bytes.Buffer``

* after buffer usage you must reset and put it to pool back:

```
buffer.Reset()
bufferPool.Put(buffer)
```

**Performance Gains**
* encoding json to bytes.buffer  
* writing bytes to bufio.Writer 
* **decoding stream to structdoes not get performance boost** 
* writtingGzip has improvements


## References
* [Go 1.5 concurent garbage collector pacing](https://docs.google.com/document/d/1wmjrocXIWTr1JxU-3EQBI6BK6KgtiFArkG47XK73xIQ/edit)
* [Proposal: Separate soft and hard heap size goal](https://github.com/golang/proposal/blob/master/design/14951-soft-heap-limit.md)
* [The Journey of Go's Garbage Collector](https://blog.golang.org/ismmkeynote)
* [Go memory ballast](https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap-26c2462549a2/)
* [The Tail at Scale](https://research.google/pubs/pub40801/)
* [Virtual Memory](https://www.geeksforgeeks.org/virtual-memory-in-operating-system/)
* [sync.Pools](https://blog.usejournal.com/why-you-should-like-sync-pool-2c7960c023ba)
* [sync.Pool Bechmarks](https://github.com/Mnwa/GoBench)
* [What is happening in Go 1.3](http://dominik.honnef.co/go-tip/2014-01-10/#syncpool)
