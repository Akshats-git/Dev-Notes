# 10 — Functions as Values: Anonymous Functions & Closures

In Go, **functions are first-class citizens**. That means a function is a value like any other. You can assign it to a variable, pass it as an argument, and return it from another function.

---

## Anonymous functions

[▶ 3:34:10](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=12850s)

An anonymous function has no name. You assign it to a variable.

```go
hypot := func(x, y float64) float64 {
	return math.Sqrt(x*x + y*y)
}

fmt.Println(hypot(5, 12))   // 13
```

Look at the shape. There is no name after `func`. And the whole thing is a value being assigned with `:=`.

After that, the variable *is* the function. You call it with `hypot(...)`.

---

## Passing functions as parameters

```go
func compute(fn func(float64, float64) float64) float64 {
	return fn(3, 4)
}
```

The parameter type here is a **function signature**: `func(float64, float64) float64`. That means two float64 in, one float64 out.

You leave out parameter *names* in a signature. Only the types matter.

Now anything matching that signature can be passed in.

```go
fmt.Println(compute(hypot))     // 5    — our anonymous function
fmt.Println(compute(math.Pow))  // 81   — a standard library function
```

`math.Pow` has the signature `func(x, y float64) float64`. It matches, so it is accepted.

This is the basis of a lot of composition in Go. You see it in middleware chains, handler wrapping, callbacks, and custom sort comparators.

### Defining and calling in one go

You can also define an anonymous function and use it on the spot. This is common when spawning goroutines.

```go
go func() {
	// runs concurrently
}()
```

Note the `()` at the end. You are defining the function *and* calling it. See [14 — Concurrency](14-concurrency.md).

---

## Closures

[▶ 3:37:34](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=13054s)

> **A note for beginners from the video.** If you know JavaScript closures, Go's work the same way. If closures are new to you, do not take this too seriously right now. You will rarely think "I should use a closure here." But closures are used all over libraries, so it is worth knowing they exist.

### What a closure is

A **closure** is a function that uses variables from **outside its own body**.

It keeps access to those variables even after the outer function has returned. The inner function "closes over" the outer function's variables. That gives it **state that survives between calls**.

### The standard example

```go
func adder() func(int) int {
	sum := 0
	return func(x int) int {
		sum += x
		return sum
	}
}
```

`adder` returns a function. That returned function can see `sum`. And `sum` stays alive after `adder` returns.

```go
func main() {
	pos, neg := adder(), adder()
	for i := 0; i < 10; i++ {
		fmt.Println(
			pos(i),
			neg(-2*i),
		)
	}
}
```

### Tracing it

| i | `pos(i)` | running `sum` (pos) | `neg(-2i)` | running `sum` (neg) |
|---|---|---|---|---|
| 0 | 0 | 0 | 0 | 0 |
| 1 | 1 | 1 | −2 | −2 |
| 2 | **3** | 3 | −6 | −6 |
| 3 | 6 | 6 | −12 | −12 |
| 4 | 10 | 10 | −20 | −20 |

The interesting part is at `i = 2`. `sum` **does not restart at 0**. It remembers the `1` from the previous call. So `1 + 2 = 3`.

That memory is the closure.

### The other key point

`pos` and `neg` come from **two separate calls to `adder()`**. So each one has **its own `sum`**. They do not interfere with each other.

Every call to the outer function creates a fresh set of closed-over variables.

---

**Next:** [11 — Methods](11-methods.md)
