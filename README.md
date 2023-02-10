# Efficient Go notes

## 01. Software Efficiency Matters
### Behind Performance
In regard to computers, performance can be summarized as: "How well is the computer doing the work that it's supposed to do"?
Many might simply reduce it's meaning to "speed". When it comes to "performance", **be specific**.  
🔑 Always clarify what someone means when they use the word "performance".  

For software: Performance = "how well software runs".
#### Three core execution elements for improvement (or sacrifice)
* Accuracy: Number of errors made
* Speed: how fast work needed to accomplish the task is done
* Efficiency: Ratio of energy delivered by a dynamic system to the energy supplied.
  * Put simply: how much energy was wasted.
```
performance = (accuracy * efficiency * speed)
```

### Common Efficiency Misconceptions
* Optimized code isn't readable  
This is often because efficiency isn't designed into our software from the beginning.  
💡It's easier to optimized readable code than make heavily optimized code readable.
* You aren't going to need it (YAGNI)  
Avoid doing extra work that isn't strictly needed for current requirements. Naturally, this doesn't mean devs should 
completely ignore efficiency, and "reasonable optimizations", should be exercised when available.
* Hardware is getting faster and cheaper  
Parkinson's Law: no matter how many resources available, demands tend to match the supply.  
Software gets slower faster than hardware becomes faster. We should also still be mindful of Moore's law and using less energy where we can.
* We can scale horizontally instead  
It's likely cheaper and simpler to implement efficiency improvements before needing to escalate scalability.
* Time to market is more important
A good balance should be found between efficient software and time to market. App performance has a direct effect on customer retention, and ultimately $$$.
Know the risk first.

### Key to Pragmatic Code Performance
Generally, focus on improving efficiency before speed as the first step. Why?
1. It's harder to make efficient software slow
2. Speed can be fragile (latency often depends on many, potentially uncontrollable factors).
3. Speed is less portable: ie moving from dev machines to production.

## 08. Benchmarking
### Microbenchmarks
Microbenchmarks focus on single, isolated functionality on a small piece of code in a process.  
Microbenchmarks help:
* Learn about runtime complexity
* A/B testing

### Go Benchmarks
To be a benchmark, a function must:
* Be in a file that ends with the `_test.go` suffix
* Start with the case-sensitive `Benchmark`. ie `BenchmarkSum` would be the benchmark func for `func Sum()`
* Have exactly one func arg of type `*testing.B`  

_Naming Conventions_:
* `BenchmarkSum` to benchmark the `Sum` func. `BenchmarkSum_withDuplicates` would also test `Sum`, but with a specific condition (note lowercase).
* `BenchmarkCalculator_Sum` would test a `Sum` method from a `Calculator` struct. And following the conditional convention above: `BenchmarkCalculator_Sum_withDuplicates`
* We can indicate input size with another suffix: `BenchmarkCalculator_Sum_10M`

❗️The size of the input will significantly change the results. There is no right answer to determine the correct input size, but we can
keep **aim for typicality**: test data should simulate production workload as much as possible.

👨🏻‍💻
**Example:**
```go
func BenchmarkSum(b *testing.B) {
	// b.ReportAllocs() // Gives number of allocations and total amount of allocated mem. Same as `-benchmem` flag.
	
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_, _ = Sum("testdata/test.2M.text")
    }
}
```
```bash
go test -run '^$' -bench '^BenchmarkSum$' # Using -run, we can ignore unit tests. Benchmark func names can be specified with the RE2 regex language.
go test -run '^$' -bench '^BenchmarkSum$' -benchtime 10s # Run as many iterations as can fit in a 10 second interval.
go test -run '^$' -bench '^BenchmarkSum$' -benchtime 100x # Run the exact amount of iterations.
go test -run '^$' -bench '^BenchmarkSum$' -benchtime 1s -count 5 # -count allows repeating the benchmarking cycle x amt of times.
```
See `go help testflag` for more options.  
💪🏻 Useful One-liner for the benchmark:
```bash
export ver=v1 && \
  go test -run '^$' -bench '^BenchmarkSum$' -benchtime 10s -count 5 \
    -cpu 4 \
    -benchmem \
    -memprofile=${ver}.mem.pprof -cpuprofile=${ver}.cpu.pprof \
  | tee ${ver}.txt
```
We can use the `-cpu` flag to control the number of CPU cores, which sets the `GOMAXPROCS` setting.  
`-benchtime`, `-cpuprofile` and `ReportAllocs` will add a slight overhead to latency, so we should turn them off for measuring ultra-low
latency.

### Benchstat
In addition to output from standard benchmark tooling, we can use [benchstat](golang.org/x/perf/cmd/benchstat) for processing/statistical analysis.  
Usage:
```
go install https://pkg.go.dev/golang.org/x/perf/cmd/benchstat@latest

benchstat v1.txt # using the output file from above one-liner.
```
Benchstat calculates the mean of all runs and the variance across runs.  
❗️This highlights the importance of running benchmarks with `-count` flag,
otherwise the variance will always be `0%`.  
We can also use benchstat to compare results:
```benchstat v1.txt v2.txt```  
benchstat tips:
* Run more tests than one with `-count` to spot noise.
* Check variance is not higher than 3-5%.
* Check significance test (p-value) for results with higher variance.

### Tips and Tricks for Microbenchmarking
Too high of variance indicates that the results might not be accurate, and we should maybe increase the `-benchtime`.

### Macrobenchmarks


## 09. Data-Driven Bottleneck Analysis
"Programmers are notoriously bad at guessing which parts of the code are the primary consumers of the resources." - Jon Louis Bentley
Knowing where the main source of latency or resource usage you want to improve is a key step to improving latency in our Go programs.

### Profiling in Go
Made possible thanks to pprof. The most basic example would be to use `runtime/pprof.Profile`. See [fd.go](profile/fd/fd.go)

### go tool pprof reports
Using the output from the above example: `go tool pprof -raw fd.pprof`  
The `-raw` flag shows us the sampling period used when capturing the profile.  
`go tool pprof -http :8080 fd.pprof`

All pprof view shows `Flat` and `Cumulative` values for locations.
* Flat: a certain node's _direct_ responsibility for resource or time usage. This tells use the source of the
potential bottleneck.
* Cumulative: a sum of _direct_ in _indirect_ contributions. Indirect means that the locations didn't
create any resource directly, but may have invoked one or more functions that did. This helps us understand what flow
is more expensive.

**Granularity**
We can specify granularity with the following `go tool pprof` flags:
* `-functions` (default)
* `-files`
* `-lines`
* `-address`  
You can also specify with the URL parameter: `?g=<GRANULARITY>`

### Capturing profile signals
There are three main methods collect profiles.
1. Programmatically triggered  
Profiles collected by calls to code. Some require a starting point, ie the CPU profiler would be started with methods
`StartCPUProfile(w io.writer)` and `StopCPUProfile()` methods.
2. Benchmark integrations  
Via `-memprofile`, `-cpuprofile`, `-blockprofile`, `-mutextprofile` flags.
3. HTTP handlers
```go
mux := http.NewServeMux()

mux.HandleFunc("/debug/pprof/", pprof.Index)
mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
mux.HandleFunc("/debug/fgprof/profile", fgprof.Handler().ServeHTTP) // third-party profiles.
```
Four ways to use profiles via HTTP:
1. Click the link for visible html page, which opens `http://<address>/debug/pprof/heap?debug=1`
2. Remove the `debug` parameter to download the file directly.
3. Open directly with pprof: `go tool pprof -http :8080 http://<address>/debug/pprof/heap`
4. Using another server to scrape the profiles.

### Common Profile Instrumention
**Heap Profile** (alloc)
To find contributors to memory allocated on the heap. It doesn't show memory allocated on stack.  
You can redirect heap profile output to `io.Writer` with `pprof.Lookup("heap").WriteTo(w, 0)`.

Default: Go records sample per every 512 KB allocated memory on the heap. This can be adjusted with `runtime.MemProfileRate` var or `GODEBUG=memprofilerate=X` env var.

Sample Types of heap profile:
* `alloc_space`: total number of allocated bytes by location on the heap since start of program
* `alloc_objects`: number of allocated memory blocks, not the actual space. Useful for finding latency bottlenecks caused by frequent allocs.
* `inuse_space`: currently allocated bytes on the heap; allocated memory minus the released memory at each location. Good for finding 
bottlenecks at a specific moment in the program. Also useful for identifying memory leaks.
* `inuse_objects`: number of allocated memory blocks on the heap. Useful to reveal amount of live objects on the heap.

**Goroutine**
