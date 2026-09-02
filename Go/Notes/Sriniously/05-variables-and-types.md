# 05 — Variables & Types

[▶ 1:09:12](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=4152s)

---

## Declaring variables

### `var` works anywhere

```go
var c, python, java bool         // package level or function level
var i int                        // zero-initialized to 0
var i, j int = 1, 2              // with initializers
var v = 10                       // type inferred from initializer
```

If several variables share a type, one type annotation at the end covers them all. If they have different types, annotate each one.

### `:=` works only inside functions

```go
func main() {
	i := 7                // declare + initialize, type inferred
	name := "akshat"
}
```

```go
k := 8   // ❌ at package level: "expected declaration, found k"
```

**`:=` only works inside a function.** Package-level declarations must use `var`.

`var v = 10` and `v := 10` mean the same thing. `:=` is just a shorter way to write it.

---

## Which form to use

There are only three cases.

| Situation | Use | Why |
|---|---|---|
| The starting value **does not matter**. It gets set later. | `var i int` | Tells the reader: starts at zero, real value comes later |
| The starting value **is known and matters** | `i := 7` | By far the most common |
| You are **outside a function** | `var c bool` | No choice. `:=` is illegal here. |

### Real-world check (fider)

Package-level variables all use `var`. They have no choice.

Inside functions you see `var err error` before a block where the error gets assigned by a later function call. The starting value is meaningless there, so `var` communicates that.

Everywhere else it is `:=`. A global search shows it is the default. Most of the time you know the starting value.

---

## Zero values

This is a core guarantee. **A Go variable is never uninitialized, undefined, or null.** Every declaration produces a real value.

| Type | Zero value |
|---|---|
| All numeric types (`int`, `float64`, `uint8`, …) | `0` |
| `bool` | `false` |
| `string` | `""` |
| All **reference types** (pointer, slice, map, channel, func) | `nil` |
| `interface` | `nil` |
| **Array** | the array with every element at its zero value. `[4]int{}` becomes `[0 0 0 0]` |
| **Struct** | the struct with every field at its zero value |

```go
type T struct {
	A int
	B string
}
var t T   // T{A: 0, B: ""}
```

This is why Go does not have a null pointer problem for value types. It is also why `if err != nil` works. See [12 — Interfaces](12-interfaces.md).

---

## The four categories of Go types

The video groups types this way. It is a useful map to keep in your head.

### 1. Basic
- **Numbers.** Integers, unsigned integers, floats, complex, plus `byte` and `rune`.
- **String**
- **Bool**

### 2. Aggregate
These are built out of other types.
- **Array.** A collection of **homogeneous** data. Everything is the same type.
- **Struct.** A collection of **heterogeneous** data. Mixed types.

### 3. Reference
These do not hold the data themselves. They point to data working in the background. There are **five of them**.
- **Pointer**
- **Slice**
- **Map**
- **Channel**
- **Function**

All five have a zero value of `nil`.

### 4. Interface
Its own category. See [12 — Interfaces](12-interfaces.md).

---

## Numeric types

```
int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr
float32 float64
complex64 complex128
byte = uint8      rune = int32
```

### What the number means

The number is the **bit width**. It decides the range.

- `int8` uses 8 bits. That is 2⁸ = 256 possible values. It is signed, so the range is **−128 to 127**.
- `uint8` uses 8 bits too. That is 256 values. It is unsigned, so the range is **0 to 255**.
- The same logic applies to 16, 32, and 64.

The signed range is not symmetric. Zero takes up a slot on the positive side. So there is one more negative value than positive.

### Plain `int` and `uint`

These are sized by your machine. They are 32 bits on a 32-bit system and 64 bits on a 64-bit system.

**The official recommendation is to default to `int` and `uint`.** Only use a sized type when you actually know the range.

### When a sized type is worth it

fider defines log levels as `uint8`:

```go
type Level uint8

const (
	DEBUG Level = iota + 1
	INFO
	WARN
	ERROR
	// ...
)
```

The range here is 1 to maybe 5. Worst case 10 or 12.

On a 64-bit machine, plain `uint` would use 64 bits for each value. That is a lot for something that never goes past a handful. `uint8` handles 0 to 255, which is plenty. It saves memory when you have many of them.

**Rule of thumb.** If you know the maximum value in advance, size the type to it. If you are handling user data or values of unknown size, use plain `int`. The performance difference is tiny and the docs recommend it.

### `byte` and `rune`

- `byte` is an alias for `uint8`. It represents the raw bytes of strings.
- `rune` is an alias for `int32`. It represents one UTF-8 character.

### Floats and complex

Floats only come in `float32` and `float64`. There is no platform-sized `float`.

Complex numbers come as `complex64` and `complex128`. They exist. You will probably never use them.

---

## Printing verbs

```go
fmt.Printf("Type: %T  Value: %v\n", x, x)
```

| Verb | Prints |
|---|---|
| `%v` | the **value** in Go's default format |
| `%T` | the **type** |
| `%s` | a string |
| `%q` | a string **with quotes** |

```go
var s string
fmt.Printf("%v\n", s)   // (prints nothing — empty string)
fmt.Printf("%q\n", s)   // ""
```

Do not memorize the verbs. You will absorb the few you actually use.

---

## Type conversion is always explicit

Go never converts types for you. Not even between numeric types that obviously fit.

```go
var k int64 = 32
var v int32 = 32

v = k              // ❌ compile error — different types
v = int32(k)       // ✅ explicit conversion
```

```go
var x, y int = 3, 4
f := math.Sqrt(float64(x*x + y*y))
z := uint(f)
fmt.Println(x, y, z)   // 3 4 5
```

The syntax is `TypeName(value)`.

You rarely have to think hard about this. These are compile-time errors, so your editor flags them as you type. You fix them while writing, not while debugging.

---

## Type inference

```go
v := 42        // int
v := 42.7      // float64
v := "hello"   // string
```

The compiler works out the type from the value you assign.

**It infers the type once, and it is permanent.**

```go
v := "hello"
v = 9   // ❌ cannot use 9 (untyped int constant) as string value
```

The type is fixed for the whole life of the variable. That is what statically typed means.

---

## Constants

```go
const Pi = 3.14
const Truth = true
const World = "世界"
```

Constants have four properties.

1. **You must give them a value at declaration.** `const x int` is an error.
2. **You cannot reassign them.** Writing `Truth = false` gives you a compile-time error.
3. **They are evaluated at compile time,** not at runtime. That is *why* the value is mandatory. The compiler has to know it.
4. **Only three types are allowed.** Number, string, and bool.

### When to use a constant

Use one any time you have a **hardcoded value**. A port number. A URL. A set of allowed values you validate against. A magic number.

Naming it does two things. It gives the value **meaning** for the reader. And it gives you a **compile-time guarantee** that it can never change.

### Real-world check (fider)

Constants are used to build an **enum**, since Go has no native enum type:

```go
const (
	RoleVisitor       Role = 1
	RoleCollaborator  Role = 2
	RoleAdministrator Role = 3
)
```

Three meaningless integers now have meaningful names. The full idiomatic enum pattern uses `iota` and a `String()` method. That comes later in the series.

---

**Next:** [06 — Control flow](06-control-flow.md)
