# Performance Optimizations

## Abstract

In this section, we will explore advanced optimization techniques, we will cover the tooling on our disposal and how to make use of it. We will learn to spot ineficiencies caused by wrong algorithms, data structures, data types and touch on the CPU cache and the efect of context switching.

## What are we optimizing for ?
* Modern processors are limited by memory latency not memory capacity
* Data alignment is less of a issue than in the past, CPUs fetch multiple cache lines at a time
* Cache misses are expensive due to CPU's speculative execution and instruction pipelines
* Vector instructions like MMX and SSE allow single instruction to be executed against multiple operands concurently, providing your code can be expressed in that form


## Benchmarking
  * Benchmarking ground rules
  * Using the testing package for benchmarking
  * Comparing benchmarks with benchstat
  * Avoiding benchmarking start up costs
  * Benchmarking allocations
  * Watch out for compiler optimisations
  * Benchmark mistakes
  * Profiling benchmarks
## Performance measurement and profiling
  * pprof
  * Types of profiles
  * One profile at at time
  * Collecting a profile
  * Analysing a profile with pprof
## Compiler optimisations
  * Escape analysis
  * Inlining
  * Dead code elimination
  * Compiler flags
  * Bounds check elimination
## Execution Tracer
  * What is the execution tracer, why do we need it?
  * Generating the profile
  * Generating a profile with runtime/pprof
  * Tracing vs Profiling
  * Using more than one CPU
  * Batching up work
  * Using workers
  * Using buffered channels
  * Mandelbrot microservice
 
## Tips and trips
  * Goroutines
  * Go uses efficient network polling for some requests
  * Watch out for IO multipliers in your application
  * Use streaming IO interfaces
  * Timeouts, timeouts, timeouts
  * Defer is expensive, or is it?
  * Avoid Finalisers
  * Minimise cgo
  * Always use the latest released version of Go

## References
* [Can we Use CPU cache](https://jquery.developreference.com/article/16680524/Is+it+possible+to+use+CPU+cache+in+Golang%3F)
* [system clock sync on KVM](https://serverfault.com/questions/132197/best-practice-for-system-clock-sync-on-kvm-host)
* [Allocation efficiency in High-performance Go Services](https://segment.com/blog/allocation-efficiency-in-high-performance-go-services/)
* [Profiling Go Programs](https://blog.golang.org/pprof)
* [benchstat](https://godoc.org/golang.org/x/perf/cmd/benchstat)
* [High Performance Go](https://dave.cheney.net/high-performance-go-workshop/dotgo-paris.html)
* [Future of computing](https://www.youtube.com/watch?v=Azt8Nc-mtKM)
