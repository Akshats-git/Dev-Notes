# 14 — Concurrency

[▶ 4:53:57](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=17637s)

This is Go's headline feature. It takes up about 58 minutes of the video.

> **Read this first.** The video is explicit that beginners should not try to master this section. Getting some of it is fine. Getting none of it is also fine. It comes with time, with reading more code, and with building projects. Do not let concurrency become the thing that makes you quit Go.

---

## Goroutines

```go
func say(s string) {
	for i := 0; i < 5; i++ {
		time.Sleep(100 * time.Millisecond)
		fmt.Println(s)
	}
}

func main() {
	go say("world")   // new goroutine
	say("hello")      // main goroutine
}
```

The output interleaves unpredictably. Something like `world hello world hello world hello`.

**`go <function call>` spawns a goroutine.** It runs at the same time as the caller.

### What a goroutine is

It is roughly like a thread. But there are important differences.

| | OS thread | Goroutine |
|---|---|---|
| **Managed by** | the operating system | **the Go scheduler** (part of the runtime) |
| **Stack size** | fixed, about 256 KB to 1 MB | **starts at 2–4 KB, grows as needed** |
| **Cost of spawning many** | high. You run out of memory fast. | low. Hundreds of thousands is normal. |

There are two takeaways. Goroutines are **managed by Go, not the OS**. And they are **very lightweight**.

### The main goroutine

**Every Go program starts with the main goroutine.** `func main()` runs inside it.

All other goroutines are spawned from it. They can spawn more themselves.

**When `main` returns, the program exits and every other goroutine dies.** They get garbage collected. Whatever they were doing never finishes. You never see the result.

```
main goroutine ──────────────────────────► return → program ends
       │                                          (all children killed)
       ├──► goroutine A ──►
       └──► goroutine B ──►
```

**So the main goroutine has to wait for the goroutines it spawned.** This need to wait is exactly why channels matter.

### Order is never guaranteed

The moment you spawn a goroutine, **you cannot know** whether it starts right away or whether the next line in the caller runs first.

Never write code that depends on it.

---

## Concurrency vs parallelism

Here is the high-level distinction. The full debate is endless.

**Parallelism is roughly a hardware property.** Tasks genuinely run at the same instant, on multiple CPU cores.

**Concurrency is roughly a software property.** You structure work so tasks run in an efficient *order* without blocking each other. The order is not specified.

Four concurrent tasks may run in any order, one at a time, interleaved. If your hardware supports parallelism, some may genuinely overlap.

But **concurrency does not mean parallelism.**

> Worth watching: Rob Pike's talks on this. He co-created Go.

---

## The philosophy: CSP

Go follows **CSP**, which stands for Communicating Sequential Processes.

> **"Don't communicate by sharing memory; share memory by communicating."**

Most languages do concurrency with **shared state** protected by locks.

Go's position is that shared state is where deadlocks, starvation, and race conditions come from. Using **communication between goroutines** as the main mechanism avoids a large class of these bugs.

Go still ships mutexes and the classic primitives. But **channels are the recommended first approach.** Reach for communication before you reach for locks.

---

## Channels

A **channel is a typed pipe.** It lets two goroutines send values to each other.

```
goroutine A  ──┤  channel  ├──►  goroutine B
```

A channel is a container. You have to say what flows through it.

```go
ch := make(chan int)      // channel of ints
```

Channels are a **reference type**, so you create them with `make`. See [05 — Variables & types](05-variables-and-types.md).

### Sending and receiving

```go
ch <- v      // send v INTO the channel
v := <-ch    // receive FROM the channel
```

The arrow points in the direction the data flows. Channel on the left means send. Channel on the right means receive.

### Worked example

```go
func sum(s []int, c chan int) {
	sum := 0
	for _, v := range s {
		sum += v
	}
	c <- sum                    // send the result back
}

func main() {
	s := []int{7, 2, 8, -9, 4, 0}

	c := make(chan int)
	go sum(s[:len(s)/2], c)     // first half in one goroutine
	go sum(s[len(s)/2:], c)     // second half in another

	x, y := <-c, <-c            // receive twice

	fmt.Println(x, y, x+y)
}
```

Here is what this does.

1. **It splits the work.** Two goroutines, each summing half the slice. This is fine because the order of the halves does not matter.
2. **It passes the channel to both** so they can report back to main.
3. **It receives twice**, collecting both results.

### The critical property

> **Receiving from a channel blocks.**

`<-c` waits until a value is available.

That is what makes this program correct. Main cannot finish early. It is parked on those two receives until both goroutines have sent.

So the channel gives you **communication and synchronization at the same time**.

Sending on an unbuffered channel blocks too, until a receiver takes the value.

---

## Buffered vs unbuffered channels

```go
ch := make(chan int)      // unbuffered
ch := make(chan int, 2)   // buffered, capacity 2
```

### Unbuffered: a meeting point

```
producer ──┤ (no storage) ├── consumer
```

There is no queue. A send blocks until a receiver is ready. A receive blocks until a sender is ready. The two goroutines have to **meet**.

**Use it when** synchronization is the point. You want the blocking. You want a handoff. You want to know the other side got it.

### Buffered: a queue

```
producer ──► [ _ | _ | _ | _ ] ──► consumer
```

The producer can push values until the buffer fills up without blocking. The consumer drains it at its own pace.

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
fmt.Println(<-ch)   // 1
fmt.Println(<-ch)   // 2
```

**Use it when** you have a producer and consumer working at different speeds. Or when you know the maximum number of values in advance.

**Neither one is better.** Buffered gives flexibility. Unbuffered gives synchronization. Pick the one you need.

---

## `close` and ranging over a channel

Say one goroutine is sending and another is receiving. **How does the receiver know when to stop waiting?**

That is what `close` is for.

```go
func fibonacci(n int, c chan int) {
	x, y := 0, 1
	for i := 0; i < n; i++ {
		c <- x
		x, y = y, x+y
	}
	close(c)                      // broadcast: no more values coming
}

func main() {
	c := make(chan int, 10)
	go fibonacci(cap(c), c)
	for i := range c {            // receives until the channel is closed
		fmt.Println(i)
	}
}
```

Output: `0 1 1 2 3 5 8 13 21 34`

### `close(ch)`

This built-in **broadcasts to every receiver**. It says sending is finished, stop blocking.

The **sender** calls it. The receiver never does.

### `for i := range ch`

Ranging over a channel does two things.

1. **It receives values one by one** as the other goroutine sends them. No manual loop. No counting.
2. **It exits automatically when the channel is closed.**

Without it you would write a `for` loop with a hardcoded count and receive N times by hand. `range` removes that boilerplate. More importantly, it removes the need to know N at all.

---

## `select`

`select` lets a goroutine wait on several channel operations at once.

```go
func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:              // if we can SEND on c
			x, y = y, x+y
		case <-quit:              // if we can RECEIVE on quit
			fmt.Println("quit")
			return
		}
	}
}

func main() {
	c := make(chan int)
	quit := make(chan int)

	go func() {                   // anonymous function, see file 10
		for i := 0; i < 10; i++ {
			fmt.Println(<-c)
		}
		quit <- 0                 // signal: we're done
	}()

	fibonacci(c, quit)
}
```

### How to read it

**`for { ... }`** is an infinite loop. See [06 — Control flow](06-control-flow.md). This is the pattern that chapter promised. You sit in a loop, listen for signals, and break out on the right one.

**`select`** checks all its cases. Whichever channel operation can go ahead, that case runs.

Each case is a **channel operation**, not a boolean condition.

A `return` inside a case exits the function, which ends the loop.

Here is the trace. The spawned goroutine receives 10 values. So for 10 iterations, `c <- x` succeeds and the Fibonacci sequence advances. Then the goroutine sends on `quit`. That case fires, prints "quit", and returns. Then `main` ends.

Output: `0 1 1 2 3 5 8 13 21 34 quit`

---

## `select` with `default`

A `default` case runs when **no other case is ready**. It makes the select non-blocking.

```go
tick := time.Tick(100 * time.Millisecond)   // fires every 100ms
boom := time.After(500 * time.Millisecond)  // fires once, after 500ms

for {
	select {
	case <-tick:
		fmt.Println("tick.")
	case <-boom:
		fmt.Println("BOOM!")
		return
	default:
		fmt.Println("    .")
		time.Sleep(50 * time.Millisecond)
	}
}
```

Output:
```
    .
    .
tick.
    .
    .
tick.
    .
    .
tick.
...
BOOM!
```

### Tracing it

`time.Tick` returns a channel that receives a value **every 100 ms**.

`time.After` returns a channel that receives **once, after 500 ms**.

Each iteration asks two questions. Is `tick` ready? Is `boom` ready? If neither is, it runs `default`, prints a dot, and sleeps 50 ms.

At 50 ms neither is ready, so default runs. At 100 ms `tick` fires. At 150 ms default runs. At 200 ms `tick` fires. And so on.

At 500 ms `boom` fires. It prints BOOM and calls `return`. That exits the loop, then exits `main`, then the program ends.

You get two dots per tick, because default sleeps 50 ms while tick arrives every 100 ms.

---

## Mutexes

Sometimes channels are not the right fit.

### The problem: concurrent map writes

```go
type SafeCounter struct {
	v map[string]int
}

func (c *SafeCounter) Inc(key string) {
	c.v[key]++
}

func main() {
	c := SafeCounter{v: make(map[string]int)}
	for i := 0; i < 1000; i++ {
		go c.Inc("somekey")
	}
	time.Sleep(time.Second)
	fmt.Println(c.v["somekey"])
}
```

You expect `1000`. What you get is:

```
fatal error: concurrent map writes
```

1000 goroutines write to the same map in an unspecified order. Go's runtime detects this and kills the program.

**This is the classic shared-state bug.** It is exactly what CSP is meant to help you avoid.

> Note the `time.Sleep`. It stands in for waiting on the goroutines. In real code you would use a channel or a **`sync.WaitGroup`**, which is covered later in the series. Sleeping is fine for a demo and wrong in production.

### The fix: `sync.Mutex`

A mutex is a **gatekeeper**. It has a `Lock()` and an `Unlock()`.

The region between them is the **critical section**. Only one goroutine can be inside at a time, no matter how many are trying.

```
             Lock()
                │
       ┌────────▼────────┐
       │ critical section│  ← only ONE goroutine in here at a time
       └────────┬────────┘
                │
            Unlock()
```

```go
type SafeCounter struct {
	mu sync.Mutex
	v  map[string]int
}

func (c *SafeCounter) Inc(key string) {
	c.mu.Lock()
	c.v[key]++
	c.mu.Unlock()
}

func (c *SafeCounter) Value(key string) int {
	c.mu.Lock()
	defer c.mu.Unlock()      // unlock on return, whatever happens
	return c.v[key]
}
```

Now you reliably get `1000`.

**Notice the `defer` in `Value`.** This is the pattern from [06 — Control flow](06-control-flow.md).

Instead of carefully placing `Unlock()` before every return path, you defer it once. It is then guaranteed to run when the function returns, including on a panic. This is the idiomatic way to unlock.

**Reads need the lock too, not just writes.** Otherwise a read can see a half-updated state.

---

## What was skipped

The video follows the Tour of Go's scope, so it admits to leaving things out.

- `sync.WaitGroup` (mentioned, saved for a later video)
- Directional channel types, meaning `chan<- int` for send-only and `<-chan int` for receive-only
- `sync.RWMutex` for read/write locks
- `context`
- Fan-in, fan-out, and other concurrency patterns
- Reflection
- Worker pools

These come later in the series. **You do not need them to write useful Go backends.**

---

## Summary

| Construct | Purpose |
|---|---|
| `go f()` | spawn a goroutine |
| `make(chan T)` | unbuffered channel — synchronization |
| `make(chan T, n)` | buffered channel — queue of capacity n |
| `ch <- v` / `<-ch` | send / receive (receiving blocks) |
| `close(ch)` | broadcast that sending is finished |
| `for v := range ch` | receive until closed |
| `select` | wait on multiple channel operations |
| `default` in select | make the select non-blocking |
| `sync.Mutex` | guard shared state when channels don't fit |

**Goroutines give you concurrency. Channels give you communication and synchronization. Mutexes are the fallback when shared state really is the right model.**

---

**Next:** [15 — Quick reference](15-quick-reference.md)
