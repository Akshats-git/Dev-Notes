# 06 — Control Flow: `for`, `if`, `switch`, `defer`

---

## `for` — the only loop

[▶ 1:37:04](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=5824s)

Go has **no `while` and no `do-while`**. It only has `for`. Every other loop shape is just `for` with parts removed.

```go
sum := 0
for i := 0; i < 10; i++ {
	sum += i
}
fmt.Println(sum)   // 45
```

There are three parts, separated by semicolons. Note there are **no parentheses** around them.

1. **Init statement.** `i := 0`. Runs once, before the loop starts.
2. **Condition.** `i < 10`. Checked before each iteration.
3. **Post statement.** `i++`. Runs after each pass through the body.

The loop variable `i` only exists inside the loop. It does not exist outside the braces.

### The cycle

```
init → [ condition → body → post ] → condition → body → post → … → condition false → exit
```

The video walks through this with the VS Code debugger. It is worth doing yourself once. Put a breakpoint inside the loop and watch `i` and `sum` change. Seeing it makes the mechanics stick in a way that reading does not.

### While loop: drop init and post

```go
sum := 1
for sum < 1000 {
	sum += sum
}
fmt.Println(sum)   // 1024
```

Keep only the condition. That is Go's `while`.

### Infinite loop: drop everything

```go
for {
	// runs forever
}
```

With no condition, the program never leaves this loop. To get out you need an explicit `break` or `return`.

```go
sum := 1
for {
	sum += sum
	if sum > 100 {
		break
	}
}
```

**An infinite loop is not a bug. It is a pattern.**

It is used heavily in concurrency. You sit in a loop, listen for signals from goroutines, and break out when the right signal arrives. See [14 — Concurrency](14-concurrency.md), especially the `select` section.

### `range`

`range` is the other way to loop. It is covered in [08 — Arrays & slices](08-arrays-and-slices.md), [09 — Maps](09-maps.md), and [14 — Concurrency](14-concurrency.md).

---

## `if`

[▶ 1:44:15](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=6255s)

```go
if x < 0 {
	return sqrt(-x) + "i"
}
```

No parentheses around the condition. The condition must be a bool.

### The init statement

Like `for`, an `if` can run a short statement before its condition.

```go
if v := math.Pow(x, n); v < lim {
	return v
} else {
	return lim
}

// v is NOT accessible here — undefined: v
```

`v` exists only inside the `if` and `else` blocks.

**This matters a lot.** It is the backbone of Go error handling. Instead of declaring a variable, calling a function, then checking, you do all three on one line.

```go
if err := doSomething(); err != nil {
	return err
}
```

This removes a lot of boilerplate. It also keeps the error variable from leaking into the surrounding scope.

### `else if` and `else`

These behave the way you expect. Conditions are checked top to bottom. The first true branch wins.

---

## `switch`

[▶ 1:47:47](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=6467s)

### Basic form

```go
switch os := runtime.GOOS; os {
case "darwin":
	fmt.Println("OS X.")
case "linux":
	fmt.Println("Linux.")
default:
	fmt.Printf("%s.\n", os)
}
```

Like `for` and `if`, `switch` takes an **optional init statement**. Here that is `os := runtime.GOOS;`. The variable `os` only exists inside the switch block.

**There is no fallthrough by default.** When a case matches, its body runs and the switch exits. You do **not** write `break`. This is the big difference from C, Java, and JavaScript. An explicit `fallthrough` keyword exists if you ever want the C behavior.

### Cases can be expressions

Cases are not limited to constants. They can be expressions evaluated at runtime.

```go
today := time.Now().Weekday()
switch time.Saturday {
case today + 0:
	fmt.Println("Today.")
case today + 1:
	fmt.Println("Tomorrow.")
// ...
}
```

### Switch with no condition

```go
t := time.Now()
switch {
case t.Hour() < 12:
	fmt.Println("Good morning!")
case t.Hour() < 17:
	fmt.Println("Good afternoon.")
default:
	fmt.Println("Good evening.")
}
```

An empty `switch` means `switch true`. Cases are checked top to bottom. The first true one runs, then the switch exits.

This is the idiomatic replacement for a long `if / else if / else if / else` chain. It reads much better.

### Type switch

There is a special form used with interfaces. See [12 — Interfaces](12-interfaces.md).

---

## `defer`

[▶ 1:50:00](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=6600s)

This concept is fairly unique to Go, and it is important.

```go
func main() {
	defer fmt.Println("world")
	fmt.Println("hello")
}
// hello
// world
```

### How it works

When you write `defer <expression>`, four things happen.

1. The arguments are **evaluated immediately**.
2. The call itself is **pushed onto a stack**. It does not run yet.
3. The rest of the function runs normally.
4. When the function **returns**, Go pops the stack and runs every deferred call. The order is **LIFO**. The last one deferred runs first.

### The stack in action

```go
func main() {
	fmt.Println("counting")
	for i := 0; i < 10; i++ {
		defer fmt.Println(i)
	}
	fmt.Println("done")
}
```

Output:
```
counting
done
9
8
7
6
5
4
3
2
1
0
```

Ten calls were pushed in order 0 to 9. They come back out in order 9 to 0. That is a stack.

---

### Why `defer` exists

There are two real reasons.

#### 1. People forget to clean up

Whenever you *start* something, you have to *stop* it. You open a file, you close it. You open a database connection, you close it.

If you forget, you leak memory. This gets bad fast inside a loop that opens many resources.

Here is the naive version:

```go
f := os.Open("file.txt")
// ... 200 lines of business logic ...
f.Close()          // easy to forget
```

With `defer`, the cleanup sits right next to the thing it cleans up:

```go
f := os.Open("file.txt")
defer f.Close()    // guaranteed, and visible right where you opened it
// ... 200 lines of business logic ...
```

This is also more readable. Anyone scanning the function sees "open, will close" in two lines next to each other.

#### 2. Panics

A **panic** is Go's rough equivalent of an exception. It is an error that stops the normal flow of execution.

Go deliberately has no exceptions. It returns errors as values instead. `panic` is reserved for things that are genuinely unrecoverable.

Without `defer`:

```
open file → run logic → 💥 panic → function exits → Close() never ran → leaked resource
```

With `defer`:

```
open file → defer Close() → run logic → 💥 panic → deferred calls run → Close() executes ✅
```

**Deferred functions run when the function returns and when it panics.** That is the guarantee. It does not matter whether the function succeeded. The cleanup happens.

---

### Real-world check (fider)

Here is a test setup:

```go
func TestSomething(t *testing.T) {
	db := setupDatabase()
	defer db.TearDown()     // second line, immediately after setup
	// ... test body ...
}
```

The database connection is torn down whether the test passes, fails, or panics.

The pattern is simple. **Acquire on one line. Defer the release on the very next line.**

---

**Next:** [07 — Pointers & structs](07-pointers-and-structs.md)
