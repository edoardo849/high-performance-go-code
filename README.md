High Performance Go Code
===

GoLab - Florence, 2017-01-20

*Go or the future of the dev*.

Build the talk in 2 parts: first part theory and then practice.

Serverless, Blockchain, Simon Wardley

Go and Serverless

Adopt some Simon Wardley maps.

Why Go is the language of the future.

Implement and benchmark the streaming IO interface to demo it live.

## Links
- [Debugging performance issues in Go programs](https://software.intel.com/en-us/blogs/2014/05/10/debugging-performance-issues-in-go-programs)
- [Dave CHeney's Profile package](https://github.com/pkg/profile)


## 2017-01-04 [High Performance Go](http://go-talks.appspot.com/github.com/davecheney/presentations/writing-high-performance-go.slide)

Dave Cheney slides. There is no one size fits all, so profiling and testing is crucial.

The primary tool for profiling Go applications is `pprof`.

Types of profiling:
- CPU profiling
- Memory profiling: mainly records when a `heap` allocation is made. Stack allocation are presumed to be free and are not tracked.


Because memory profiling is sample based and because it records allocation and not use, the total use of memory of the application is hard to determine. Moreoveer, profiling itself is not free: definitely do not enable more than one profile at a time.

Comparing the benchmarks.

Created project /high-perf-go/test-pkg following example here: http://go-talks.appspot.com/github.com/davecheney/presentations/writing-high-performance-go.slide#17 .

Had to install `benchstat`:

```bash

go get rsc.io/benchstat
go test -bench=. -count=5 > old.txt
go test -bench=. -count=5 > new.txt
benchstat old.txt new.txt

```

Go is garbage collected by design. This feature will not change. The performance of Go programs are often determined by their interaction with the garbage collector.

Strategies for lowering memory usage if garbage collector performance is a bottleneck.

Reducing allocations:

```go

// Parenthesys omitted on return statement because of formatting bug of neovim.

func (r *Reader) Read()  ([]byte, error)
func (r *Reader) Read(buf []byte) (int, error)

```

The first function will always allocate a buffer, while the second one fills the buffer, allowing the caller to reuse it.

most IO is done with bytes, not strings. Converting between the 2 generates garbage.

- avoid []byte to string conversion: either stick with `string` or `byte` as representation.
- the `bytes` package as many of the same operations as the `string` package anyway.
- `strings` uses the same assembly primitives as the `bytes package`


If you have a `[]byte` as a map key:

```go

var m map[string]string

// you can do
v, ok := m[string(bytes)]


// however, this won't trigger the compiler optimisation
key := string(bytes)
v, ok := m[key]

```

- avoid string concatenation.


```go

// avoid
s := request.ID
s += " " + client.Address().String()
return s

// prefer

b := make([]byte, 0, 40) // guess
b = append(b, reques.ID...)
b = append(b, client.Address().String()...)

return string(b)

```

Strings are immutable, therefore concatenation between 2 strings generates a third.

- pre-allocate slices capacities.
- Never start a goroutine without knowing how it will stop.
- Use streaming IO interfaces: do not read data into a `[]byte` and pass it around.
- Use `io.Reader` and `io.Writer` to construct processing pipelines to cap the amount of memory in use per request. In this case, consider implementing the `io.ReaderFrom` / `io.WriterTo` instead of `io.Copy` because those are more efficient and avoid copying memory into a temporary buffer.
- Interfaces `io.Reader` and io.Writer are not buffered: use bufio.NewReader(r) and bufio.NewWriter(w) to get a buffered reader and writer.
- always implement timeouts for IO operations
- any operation on a `*os.File` don't use Go polling and will therefore consume one OS thread. To manage lots of IO operations, use a pool of worker goroutines or a buffered channel as a semaphore.
- avoid IO in the context of a request-- do not let your user wait for your disk subsystem to write to disk.
- shorter code is smaller code; which is important for CPU caching


Why do we need high-performance programs anyway?

- MicroServices architectures (heavy on network I/O even with protobuf)
- IoT devices
- Serverless
- Mobile Devices looking at full convergence
