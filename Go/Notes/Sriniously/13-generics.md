# 13 — Generics

[▶ 4:46:40](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=17200s)

Generics are new. They arrived in **Go 1.18**. They were not in the original language.

---

## The problem they solve

Without generics, you write the same function once per type.

```go
func IndexInt(s []int, x int) int {
	for i, v := range s {
		if v == x {
			return i
		}
	}
	return -1
}

func IndexString(s []string, x string) int {
	for i, v := range s {
		if v == x {
			return i
		}
	}
	return -1
}
```

**The bodies are identical.** The only difference is `int` versus `string`.

Static typing forces you to duplicate the implementation. It is pure boilerplate.

---

## Type parameters

Generics let you pass **types as parameters**, alongside the values you already pass.

```go
func Index[T comparable](s []T, x T) int {
	for i, v := range s {
		if v == x {
			return i
		}
	}
	return -1
}
```

There are three changes from the non-generic version.

```
func Index[T comparable](s []T, x T) int
          └─────┬──────┘ └──────┬──────┘
       type parameter list   regular parameters,
       (square brackets)     now using T
```

1. **`[T comparable]`** is the type parameter list. It goes in **square brackets**, before the regular parameters. It declares a type parameter named `T`.
2. **`comparable`** is the **constraint** on `T`. `T` is not "any type." It must support `==` and `!=`. That is required here because the body does `v == x`.
3. **`s []T, x T`** are the regular parameters. They now use `T`. Go binds `T` to a concrete type at compile time.

The return type can use `T` too.

---

## Calling it

```go
si := []int{10, 20, 15, 3}
fmt.Println(Index(si, 15))        // 2

ss := []string{"foo", "bar", "baz"}
fmt.Println(Index(ss, "hello"))   // -1
```

**You do not write the type argument.** The compiler works out `T` from the arguments. It sees that `ss` is `[]string` and `"hello"` is a `string`, so `T = string`.

You can write it explicitly when inference cannot figure it out:

```go
Index[string](ss, "hello")
```

**One function now handles both types.** There is one implementation to maintain instead of two.

---

## Constraints

The constraint is what makes a type parameter useful instead of useless.

A constraint is written as an interface. It tells the compiler which operations are legal on `T`.

- **`comparable`** supports `==` and `!=`. You need it for equality checks and for map keys.
- **`any`** is no constraint at all. You can pass the value around but you can do almost nothing with it.
- **Custom constraints** let you list allowed types. For example, a numeric constraint that allows all int and float kinds, so you can use `+` and `<`.

Without a constraint, the compiler has no idea whether `v == x` is legal for whatever `T` turns out to be. That is why it is required.

---

## How much this matters right now

> "You might not use generics as extensively when you're starting out, but you will definitely see different implementations and use cases of them."

Generics are worth **recognizing** more than reaching for.

Most application-level backend code does not need them. You will meet them in libraries and utility packages. The `slices` and `maps` standard library packages are built on them.

**What to remember:**

- Type parameters go in square brackets, before the value parameters.
- They come with constraints.
- They can appear in parameter types and in return types.
- The point is to remove duplicate implementations that only differ by type.
- Use them when two functions have the **same body** and differ only in types.

Generics also work on type declarations, not just functions. That gives you generic types like a `List[T]` linked list. The Tour covers this briefly. It is the same idea applied to types.

---

**Next:** [14 — Concurrency](14-concurrency.md)
