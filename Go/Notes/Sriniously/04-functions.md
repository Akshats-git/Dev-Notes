# 04 — Functions

[▶ 1:00:25](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=3625s)

---

## The structure

```go
func add(x int, y int) int {
	return x + y
}
```

| Part | Rule |
|---|---|
| `func` | the keyword every function starts with |
| `add` | the name |
| `(x int, y int)` | parameters. **Name first, type second.** Comma separated. |
| `int` | the return type. You **must** declare it. |
| `{ ... }` | opening brace goes on the **same line** |

Note the name-then-type order. It is the reverse of C and Java. Go is consistent about it everywhere. Variables, struct fields, and parameters all work this way.

### Static typing is enforced here

```go
add("hello", 7)
// cannot use "hello" (untyped string constant) as int value in argument to add
```

Go is statically typed. Argument types are checked when you compile, not when you run.

---

## Shortening parameters of the same type

```go
func add(x, y int) int { ... }   // same as (x int, y int)
```

If parameters sit next to each other and share a type, you can drop the type from all but the last one.

> **The video prefers the long form** `(x int, y int)`. On first read it says outright that x is an int and y is an int. That is slightly more readable and leaves less room for a misread. This is a preference, not a rule.

---

## Multiple return values

A Go function can return more than one value. You will use this constantly. It is the foundation of Go's error handling.

```go
func swap(x, y string) (string, string) {
	return y, x
}

func main() {
	a, b := swap("hello", "world")
	fmt.Println(a, b)   // world hello
}
```

Three things to remember here.

- More than one return value means you **need parentheses** around the return types.
- Exactly one return value means no parentheses. Just `func f() string`.
- You receive the values with a comma-separated list of variables. They come back in order.

---

## Named return values (and why to skip them)

Go lets you name the return values in the signature.

```go
func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return              // "naked return" — implicitly returns x, y
}
```

Here is what happens.

`x` and `y` are **declared and set to zero** at the top of the function body. This happens before the first statement runs.

That is why the body uses `x =` to assign, not `x :=` to declare.

A bare `return` sends back whatever `x` and `y` hold at that moment.

### The recommendation: do not use this

> "I have never used it. I don't want to use it."

The problem is readability in long functions.

You reach a bare `return` at the bottom of a function. You have no idea what it returns. So you scroll up to the signature. You find the names. Then you trace those variables through the whole body.

It makes things more complicated than they need to be. It is also easy to get wrong.

Write it explicitly instead:

```go
func split(sum int) (int, int) {
	x := sum * 4 / 9
	y := sum - x
	return x, y
}
```

Same result. And now the return statement tells you exactly what comes back.

**You should still know naked returns exist.** You will read them in other people's code. Just do not write them.

---

## Related topics

Functions are **first-class values** in Go. You can assign them to variables, pass them as arguments, and return them. See [10 — Functions as values](10-functions-as-values.md).

A function attached to a type is a **method**. See [11 — Methods](11-methods.md).

A function's *signature* is what satisfies an **interface**. See [12 — Interfaces](12-interfaces.md).

Variadic functions take any number of arguments. They look like `func f(args ...int)`. `append` and `fmt.Println` are variadic. See [08 — Arrays & slices](08-arrays-and-slices.md).

---

**Next:** [05 — Variables & types](05-variables-and-types.md)
