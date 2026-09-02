# 11 — Methods

[▶ 3:42:21](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=13341s)

A **method** is a function attached to a type. Go has no classes. Methods are how you give a type behavior.

**You can define a method on any type.** Structs, ints, strings, slices, anything. The only condition is that the type must be defined in the same package.

---

## The syntax: receivers

```go
type Vertex struct {
	X, Y float64
}

func (v Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}
```

The difference from a normal function is the extra parameter list **before the name**.

```
func (v Vertex) Abs() float64
     └────┬────┘
      the receiver
```

The receiver's type is what the method gets attached to. This one declares that `Abs` is a method on `Vertex`.

Inside the body, `v` is a **concrete instance** of the struct. You reach its fields through it.

### Calling it

```go
v := Vertex{3, 4}
fmt.Println(v.Abs())   // 5
```

When you call `v.Abs()`, `v` gets passed into the receiver.

**Convention.** Receiver names are short, like `v`, `u`, or `s`. You type them constantly. It is not worth arguing about.

---

## Methods on non-struct types

```go
type MyFloat float64

func (f MyFloat) Abs() float64 {
	if f < 0 {
		return float64(-f)
	}
	return float64(f)
}

f := MyFloat(-math.Sqrt2)
fmt.Println(f.Abs())
```

`type MyFloat float64` defines a **custom type**. Its underlying type is `float64`.

You can hang methods on it. This is the mechanism behind things like `type Role int` with a `Role.String()` method.

---

## Value receiver vs pointer receiver

```go
func (v Vertex) Abs() float64    { ... }   // VALUE receiver
func (v *Vertex) Scale(f float64) { ... }  // POINTER receiver
```

### The difference

| Receiver | What gets passed | Do changes stick? |
|---|---|---|
| **Value** `(v Vertex)` | a **copy** of the instance | ❌ No |
| **Pointer** `(v *Vertex)` | the **address** of the instance | ✅ Yes |

```go
func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

v := Vertex{3, 4}
v.Scale(10)
fmt.Println(v.Abs())   // 50, not 5 — Scale actually mutated v
```

Change `*Vertex` to `Vertex` in `Scale` and the answer goes back to `5`. The change happened to a copy, and the copy was thrown away.

---

## Go inserts `&` and `*` for you

With a **function**, the types have to match exactly.

```go
func ScaleFunc(v *Vertex, f float64) { ... }

v := Vertex{3, 4}
ScaleFunc(v, 10)    // ❌ cannot use v (Vertex) as *Vertex
ScaleFunc(&v, 10)   // ✅ you must write the &
```

With a **method**, Go adds it for you.

```go
v := Vertex{3, 4}
v.Scale(10)   // ✅ works — Go rewrites this as (&v).Scale(10)
```

It works in both directions. If you have a pointer and call a value-receiver method, Go dereferences it. So `p.Abs()` becomes `(*p).Abs()`.

The benefit is readability. You write `p.Scale(10)` instead of `(&p).Scale(10)` all over your code. You stop having to check whether the receiver signature matches what you are holding.

---

## Choosing a receiver: three rules

> These three rules are the practical takeaway of this whole section.

### 1. Are you changing the instance? Use a **pointer receiver**.

If the method modifies the instance and you want that change to stick, you need a pointer. A value receiver modifies a copy that is thrown away immediately.

### 2. Is the struct large? Use a **pointer receiver**.

Every call with a value receiver **copies the whole struct**.

For a struct with 50 or 100 fields, including slices, nested structs, and maps, that copying is real work on every single call. It is not catastrophic. But it is avoidable. A pointer copies one address.

### 3. Does any other method on this type use a pointer? Then **use pointers everywhere**.

**Do not mix value and pointer receivers on the same type.** Pick one and apply it to every method.

Mixing them causes subtle bugs. As [12 — Interfaces](12-interfaces.md) shows, it also breaks interface satisfaction in confusing ways.

If even one method needs to modify the instance, make all of them pointer receivers.

**If none of these apply, use a value receiver.**

---

## Why use methods instead of functions

[▶ ~3:57:00](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=14220s)

The result is the same either way. So why bother?

### The real-world argument (fider)

fider defines a `User` struct with methods on it.

```go
func (u *User) HasProvider(provider string) bool {
	for _, p := range u.Providers {
		if p.Name == provider {
			return true
		}
	}
	return false
}

func (u *User) IsCollaborator() bool  { ... }
func (u *User) IsAdministrator() bool { ... }
```

Now look at the alternative with plain functions.

```go
func UserHasProvider(u *User, provider string) bool
func IsUserCollaborator(u *User) bool
func IsUserAdministrator(u *User) bool
```

Notice what you are forced to do. You have to **put "user" into every function name**. There is no other way to signal what the function is for.

A bare `HasProvider()` floating in a package means nothing to a new engineer. There is no way to tell what it is meant to work with.

Methods give you that scope for free.

```go
u.HasProvider("google")
u.IsCollaborator()
u.IsAdministrator()
```

The type is right there in the call. Names get shorter. The scope is obvious. And the list of methods reads as **documentation of what a `User` can do**.

### The other benefit

If you come from an object-oriented language, this maps onto class methods. Methods let you say "this type has these capabilities" as one coherent group.

More importantly, **methods are what satisfy interfaces**. That is where Go's real power lives. See [12 — Interfaces](12-interfaces.md).

### A note on that fider example

`HasProvider` does not change anything, yet it uses a pointer receiver. Why?

We cannot know for sure without reading the whole codebase. But the likely reasons are rule 2 (`User` is a large struct loaded from a database) or rule 3 (some *other* method on `User` does mutate, so consistency forces pointers everywhere).

---

**Next:** [12 — Interfaces](12-interfaces.md)
