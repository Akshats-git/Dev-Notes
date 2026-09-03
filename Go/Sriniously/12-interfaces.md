# 12 — Interfaces

[▶ 4:02:14](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=14534s)

This is the deepest idea in Go's design. You will meet it everywhere in the standard library. It takes up about 45 minutes of the video.

> The video's advice: do not try to master interfaces on the first pass. Get the basics and move on. Understanding builds up as you read and write more code.

---

## What an interface is

An interface is a type whose definition is a **set of method signatures**.

```go
type Abser interface {
	Abs() float64
}
```

That is the whole syntax. Interfaces contain **method signatures and nothing else**.

An interface describes **behavior**. It says: anything with these methods.

---

## Implicit satisfaction

This is the defining feature of Go interfaces.

```go
type MyFloat float64
func (f MyFloat) Abs() float64 { ... }

type Vertex struct{ X, Y float64 }
func (v *Vertex) Abs() float64 { ... }

func main() {
	var a Abser

	f := MyFloat(-math.Sqrt2)
	a = f          // ✅ MyFloat has Abs() float64

	v := Vertex{3, 4}
	a = &v         // ✅ *Vertex has Abs() float64

	a = v          // ❌ compile error
	// Vertex does not implement Abser
	// (method Abs has pointer receiver)
}
```

**There is no `implements` keyword.** There is no declaration and no registration.

A type satisfies an interface *automatically* if it has the required methods with matching signatures.

This is different from Java, C#, and TypeScript, where you declare your intent. In Go the compiler discovers the relationship. A type can satisfy an interface that was written years later by someone who never saw that type.

Recall from [05 — Variables & types](05-variables-and-types.md) that Go normally refuses to assign one type to another without explicit conversion. Interfaces are the exception. But that is only because the type really does provide the required behavior.

### ⚠️ The receiver gotcha

Notice that `a = v` fails while `a = &v` works.

Why? Because `Abs` was declared with a **pointer receiver**, `(v *Vertex)`. Only `*Vertex` has that method. A plain `Vertex` value does not.

This is a concrete reason for the rule in [11 — Methods](11-methods.md). **Do not mix value and pointer receivers on one type.** Mixing them produces exactly this kind of confusing error.

---

## What an interface value holds

An interface value is a **container with two fields**.

```
┌─────────────────────────┐
│ INTERFACE VALUE         │
│   dynamic type          │
│   dynamic value         │
└─────────────────────────┘
```

```go
type I interface{ M() }

type T struct{ S string }
func (t T) M() { fmt.Println(t.S) }

var i I           // both fields empty → i is nil
i = T{"hello"}    // dynamic type = T, dynamic value = T{"hello"}
i.M()             // hello
```

Before you assign anything, the container is **empty. The zero value of an interface is `nil`.**

The compiler will let you *write* `i.M()`, since the interface declares that method. But at runtime:

```go
var i I
i.M()   // 💥 panic: runtime error: invalid memory address or nil pointer dereference
```

**An interface gives you behavior, not data.** You have to give it a concrete implementation before it is usable. Call it before that and the program panics.

---

## The empty interface

```go
interface{}
```

It has zero methods.

Satisfaction means "has all the required methods." There are no required methods here. So **every type in Go satisfies the empty interface.**

```go
var i interface{}
i = 42
i = "hello"
i = []int{1, 2, 3}   // all fine
```

### `any`

Go 1.18 added an alias:

```go
type any = interface{}
```

It is purely a readability shortcut. The meaning is identical.

### Where you have already used it

```go
func Println(a ...any) (n int, err error)
```

`fmt.Println` and its relatives are **variadic functions taking `...any`**. That means any number of arguments of any type. This is how they accept whatever you throw at them.

---

## Type assertion

Once a value is inside an empty interface, Go's type system will not let you use it.

```go
var i any = "hello"
i = i + 2      // ❌ invalid operation: mismatched types interface{} and int
len(i)         // ❌ invalid argument: i (variable of type interface{}) for len
```

You cannot do arithmetic on it. You cannot take its length. You have to **assert** it back to a concrete type first.

```go
s := i.(string)
fmt.Println(s)   // hello
```

### The two-value form (safe)

```go
s, ok := i.(string)    // "hello", true
f, ok := i.(float64)   // 0, false — no panic
```

`ok` tells you whether the assertion worked. On failure you get the zero value.

### The one-value form (unsafe)

```go
f := i.(float64)   // 💥 panic: interface conversion: interface {} is string, not float64
```

**If you leave out the `ok` variable, a failed assertion panics.** This is not a compile error. It is a runtime crash.

> **Use the two-value form** unless you are certain of the type.

---

## Type switch

Asserting one type at a time gets tedious. A type switch handles a list of them.

```go
func do(i any) {
	switch v := i.(type) {
	case int:
		fmt.Printf("Twice %v is %v\n", v, v*2)
	case string:
		fmt.Printf("%q is %v bytes long\n", v, len(v))
	default:
		fmt.Printf("I don't know about type %T!\n", v)
	}
}

do(21)      // Twice 21 is 42
do("hello") // "hello" is 5 bytes long
do(true)    // I don't know about type bool!
```

The special syntax is **`i.(type)`**. It is only legal inside a switch header.

Inside each case, `v` already has that case's concrete type. That is why `v*2` works in the `int` case and `len(v)` works in the `string` case.

You will see this pattern anywhere `any` is involved.

---

## `Stringer`

This is a standard library interface worth knowing.

```go
// package fmt
type Stringer interface {
	String() string
}
```

**Any type with a `String() string` method is a Stringer.** That is the whole contract.

```go
type Person struct {
	Name string
	Age  int
}

func (p Person) String() string {
	return fmt.Sprintf("%v (%v years)", p.Name, p.Age)
}

a := Person{"Arthur Dent", 42}
fmt.Println(a)   // Arthur Dent (42 years)
```

Comment out the `String()` method and the output goes back to Go's default struct format, `{Arthur Dent 42}`.

### How it works

`Println`, `Printf`, and `Sprintf` all take `...any`.

Internally they check whether each argument satisfies `Stringer`, meaning it has a `String()` method. If it does, they call it. If not, they fall back to the default format.

This is implicit satisfaction paying off. You control how your type prints everywhere, just by adding one method. No registration. No inheritance.

> **Naming convention.** Interfaces usually end in **`-er`**. You see `Stringer`, `Reader`, `Writer`, and `Handler`. Interfaces describe *behavior*, and the `-er` suffix says "the thing that does X."

Note `fmt.Sprintf` in the example above. It **formats and returns** a string instead of printing it.

---

## `error`

This is the interface behind all Go error handling.

```go
// builtin
type error interface {
	Error() string
}
```

**Any type with an `Error() string` method is an error.** Same mechanism as `Stringer`.

```go
type MyError struct {
	When time.Time
	What string
}

func (e *MyError) Error() string {
	return fmt.Sprintf("at %v, %s", e.When, e.What)
}

func run() error {                       // returns the INTERFACE
	return &MyError{
		time.Now(),
		"it didn't work",
	}                                    // returns a CONCRETE type
}

func main() {
	if err := run(); err != nil {
		fmt.Println(err)
	}
}
```

### Why return `error` instead of `*MyError`

This is the whole reason `if err != nil` works.

The zero value of an interface is **`nil`**. So "no error" is just returning `nil`.

And "an error happened" is any concrete type that satisfies `error`.

The caller does not need to know *which* error type came back. It only checks:

```go
if err != nil {
	// failure path
}
// success path
```

One check works against every error type that exists.

This is the foundation of Go's error handling. Errors are ordinary return values. There are no exceptions and no try/catch.

---

## Interfaces in real backends

[▶ ~4:32:00](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=16320s)

This is the most valuable practical use of interfaces in backend Go.

### The typical layers

```
┌──────────────────┐
│   Controller     │  HTTP: parse body/query params, validate,
│                  │  call service, write response + status code
├──────────────────┤
│    Service       │  Business logic: orchestrates repository calls,
│                  │  sends emails, schedules tasks
├──────────────────┤
│   Repository     │  Database: queries, inserts, updates, deletes
└──────────────────┘
```

Some systems skip the service layer and let controllers call repositories directly. Terminology varies. Three layers is typical.

### The pattern (incubator-answer)

```go
// declared in the SERVICE package
type AuthRepo interface {
	GetUserCacheInfo(ctx context.Context, accessToken string) (*entity.UserCacheInfo, error)
	SetUserCacheInfo(ctx context.Context, ...) error
	// ...
}

type AuthService struct {
	authRepo AuthRepo
}

func NewAuthService(authRepo AuthRepo) *AuthService {
	return &AuthService{authRepo: authRepo}
}
```

The service **does not import a concrete repository struct**. It takes an **interface** as a constructor parameter.

That is **dependency injection**.

### Where to declare interfaces

> **Declare the interface where it is *used*, not where it is implemented.**

The consumer, which is the service here, defines what it needs. The producer, which is the repository, just happens to satisfy it.

Not the other way around. The producer should not define and export an interface for its consumers.

There are valid exceptions. But this rule holds most of the time. It is the opposite of the Java instinct.

### Why this matters: testing without a database

The payoff shows up when you write **unit tests**.

You do not want to start a Docker container and hit a real database just to test business logic. It is too slow and too fragile.

Because `AuthService` accepts an interface, you get two paths:

```
Production code path:
    NewAuthService(realAuthRepo)   → actual SQL queries

Test code path:
    NewAuthService(mockAuthRepo)   → returns hardcoded values / in-memory slices
```

Both satisfy `AuthRepo`, so both are accepted. The compiler does not care which one it is. `AuthService` only cares that the methods exist.

Your mock is just a struct with the same method set, returning canned data.

**You do not need a mocking library.** Go's implicit interface satisfaction gives you this for free. That is a big reduction in test boilerplate compared to languages that need mock frameworks.

---

## Interfaces elsewhere in the standard library

You will see `io.Reader` and `io.Writer`, which are the backbone of all I/O. You will also see `os.File`, `http.Handler`, `sort.Interface`, and many more.

Because satisfaction is implicit, a lot of composition comes for free. Any type with a `Read([]byte) (int, error)` method works everywhere an `io.Reader` is expected.

---

**Next:** [13 — Generics](13-generics.md)
