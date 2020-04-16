# The GO Scheduler

## Abstract 

In this session we will learn how the go schduler works. How go routines are organized, how they are managed, How they communicate and what is context switching. Last but not least we will see how program's perofrmance is impacted by the schduler's decisions.


## goroutines
Go provides concurancency as first class sitizen and it is baked into it's very fabrik. 
* a goroutine is user space a lightweight threads
* run in the same address space, so access to shared memory must be synchronized
* similar to the kernel threads managed by the OS, but managed by the Go runtime

Memory footprint
* Pro: default of 2 KB vs 8 KB of a standard thread
   * Growable stacks
* Con: state tracking overhead


Fast creation, descturction, context switching
* Stack Performance
  * Split stack O(1) per function call, penalizing fast ops, ~2 ns
  * Growable Stack O(N) cost per function call, penalizing expensive ops, ~60 ns
* goroutine switches  = ~ tens of ns, thread switches = ~ a ms

## The scheduler

A Go runtime component, which manages and multiplexes goroutines on top of OS kernel threads

**The MPG Model**
* OS Threads -> M
* Logical Processors -> P
* goroutines -> G
**The main goals**:
* use a small number of kernel threads
  * they are expensive
  * the kurnel has no clue waht we are running
* support high concurency
  * we want to have a very high nnumber of concurent goroutines
* levarage parallelism
  * on a N-core system, Go programs should be able to run N goroutines in parallel

**Limitations**
* FIFO runqueues 
  * priorities are not handled ok
  * linux scheduler for example has a priority queue
* basic fairness
  * no strong preemtion  -> Non-cooperative goroutine preemption
* not aware of system topology -> no real locality
  * NUMA-aware scheduler
* LIFO runqueues for better cache utilization



**When does it do it ?**

At operations that should affect the goroutine exections.
The runtime causes a switch into the scheduler under the hood and the scheduler may decide a different goroutine is to be executed on the current thread.

**How it handles the following?**

* goroutine creation
* goroutine blocking 
* blocking system calls

### How the multiplexing happens
* distribution of goroutines on kernel threads
* how to multiplex goroutines onto kernel threads
* when to create create kernel threads
* how many kernel threads we need 

#### Runqueues

* heap allocated FIFO queues are used to track ready to run goroutines
* reusing the Kernel Threads 
  * to high will give us contention
  * to low and we won't use all CPU cores
  * GOMAXPROCS env variable  - > count (CPU cores)
* locking e.g. mutex / contention of the runqueue by threads trying to get work
* work stealing
* number of runqueues/ global runques
* organization of the distributed runqueues and sync
* dealing with blocking system calls 
  * transfering long blocked threads runqueue to other threads
* preemption
  * cooperative preemption
    * There are calls to the scheduler when specific calls are made such as function calls, loops, and other (checkpoints), //go:noinline function calls
    * go
  * sysmon background thread to detect long running goroutines
     * all > **10ms** get's unscheduled when possible
* send the hungry runnning goroutines on the gloab runqueue
* global runqueue is low prio runqueue
* As of go 1.14 goroutines are asynchronously preemptible (not supported on windows/arm, darwin/arm, js/wasm, and plan9/*.)
  * Loops without function calls no longer delay GC

#### Context Switching

The process of putting the curently running goroutine (G)'s stack in the Process RunQ and scheduling another goroutine on the Kernel Thread.
* quite fast actually
* impacts the data caches and performance
* we don't block the OS thread

#### State tracking
* blocked
* runnable
* How about waiting ?
* How about preempted ?

## Channels

Don't communicat by sharing memmory, share memmory by comunicating - Rob Pike

* Store and pass values between goroutines
* provide FIFO semantics
* goroutine safe 
* can cause a goroutine to block and unblock

### Making channels
 
#### Buffered channels
* All properties of buffers +
* stored up to capacity elements  (FIFO semantics)

```ch:= make(chan Task, 3)```

* allocates hchan struct on the heap
* returns a pointer
* should pass by value


**Internal structure -hchan**
* circular queue as buffer a.k.a. ring buffer
* send index
* receive Index
* mutex lock



#### NonBuffered channel
```ch:= make(chan Task)```


### Send/Receive
* tight integration with the runtimethe scheduler

**Sequences of sending a value**
* acuire lock
* copy value to buffer
* update send index
* release lock

**Sequences of receiving a value**
* acuire lock
* copy value from buffer
* update recived index
* release lock

**Runtime optimizations**
* hchan has a pointers to waiting senders and receivers goroutines
  * also pointers to variables in their memory  
  * sometimes sender's goroutine will bypass buffer and write directly to the reciver's memory( heap/stack )
    * reciever does not need to lock and etc
    * one less copy
    * happens when a reciever is scheduled before a sender exists
* integrations withthe runtime allow
  * sender to unblock receiver/s
  * reviever to unblock sender
  * bypass the runQ sometimes

**select statement with a channel receive**
* makes calls to scheduler 
  * gopark
    * context switch is made -> associacion of Kernel/OS Thread (M) and goroutine's context is removed and pushed to the runq

**The secrets to channels**
* goroutine safe
  * internal mutex for sync
* sores values FIFO
  * copy values in internal ring buffer
* can cause goroutines to pause/resume
  * hcchan sudog queues
  * calls into the scheduler
    * gopark, goready

**NonBuffered chans**
* always act as the "direct send" 
  * receive first -> sender wrties to reciever's memory
  * sender first -> receiver recieves directly from the sudog 

**select - general case**
* all channels are locked
* sudog is put in the sendq/recvq queues of all channels
* channels unlocked, anmd the select-ing G is pasued
* CAS operations so there's one winning case
* resuming mirrors the pausing sequence

### Design considerations
* simplicity lock vs lockless implementation
  ```The performance improvement does not materialize from the air, it comes with complexity increase" - dvoykov```

* performance
  * lessless wake-up path
  * fewer memory copies
  * OS thread remains runnning

* implications
  * memory management
  * stack-shrinking

## Pitfals
**Nothing comes for free**


* Closure variables are evaluated when the goroutine runs! closures and goroutines may lead to nondeterministic outcome. For example 

```
for i:=0; i<=9; i++ {
  go func(){
    fmt.Println(i)
  }()
}
```
VS
```
for i:=0; i<=9; i++ {
  go func(i int){
    fmt.Println(i)
  }(i)
}
```
* Go runtime will try not to interupt computation
* Go runtime is context aware and knows what will cause the Kernel Thread to block
* Scheduler will try to optimize 
* Preemtion sucks
  * Has impact on GC, as it needs to do **Stop the world** once in a while
  * what if it can't 
```
go func () {
  for i=0; i<=255; i++{
  }
  
}()
fmt.Println("Dropping mic")
//Yeild execution to force executing other goroutines
runtime.Gosched()
runtime.GC()
fmt.Println("Done")
```
* Stoping a goroutine 
* * wrapping in a second goroutine and signaling status or waiting of timeout in parent is a path to leacking go routines
```
go func() {
  h.Handler.ServeHTTP(tw, r)
  // signal done
}()

select {
  case <-done:
  //handle http stuff
  case <-timeout:
  //write error
}

```

## Refferences

* [The scheduler saga](https://www.youtube.com/watch?v=YHRO5WQGh0k&t=135s)
* [The Dark side of routines](https://www.youtube.com/watch?v=4CrL3Ygh7S0)
* [Understanding channels](https://www.youtube.com/watch?v=KBZlN0izeiY)
* [Go Scheduler - lightwieght concurency](https://youtu.be/-K11rY57K7k)
