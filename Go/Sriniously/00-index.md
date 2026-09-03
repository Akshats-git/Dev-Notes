# The LAST Go basics video you will ever watch — Notes

**Source:** [Sriniously — "The LAST Go basics video you will ever watch"](https://www.youtube.com/watch?v=tgGNwG_UxFo)
**Length:** 5h 54m 50s
**Series:** *Production Grade Go from Scratch* (video 1 of the playlist)
**Companion resource:** [A Tour of Go](https://go.dev/tour/welcome/1) — the video walks through the Tour's examples

**Open-source repos used for real-world examples:**
- [getfider/fider](https://github.com/getfider/fider) — feedback management platform
- [apache/incubator-answer](https://github.com/apache/incubator-answer) — Q&A platform

---

## How to use these notes

The video gives one piece of advice up front. Do the Tour of Go yourself first. It takes 2–3 hours. Then watch the video or read these notes.

The point of the first pass is to get a feel for the syntax. You are not supposed to understand everything yet.

These notes are split by topic. Read them in order the first time. After that, use [15-quick-reference.md](15-quick-reference.md) as a cheat sheet.

---

## Contents

| # | File | Covers | Video time |
|---|------|--------|-----------|
| 01 | [Mindset & approach](01-mindset-and-approach.md) | Why this video exists, how to learn Go, what to skip | [0:00](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=0s) |
| 02 | [Modules & project setup](02-modules-and-project-setup.md) | `go.mod`, `main.go`, UTF-8, `go build` vs `go run` | [27:51](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=1671s) |
| 03 | [Packages & visibility](03-packages-and-visibility.md) | Package rules, naming conventions, exported identifiers | [51:48](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=3108s) |
| 04 | [Functions](04-functions.md) | Signatures, multiple returns, named returns | [1:00:25](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=3625s) |
| 05 | [Variables & types](05-variables-and-types.md) | `var` vs `:=`, data type taxonomy, zero values, conversion, constants | [1:09:12](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=4152s) |
| 06 | [Control flow](06-control-flow.md) | `for`, `if`, `switch`, `defer` | [1:37:04](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=5824s) |
| 07 | [Pointers & structs](07-pointers-and-structs.md) | Reference types, pointers, structs, struct tags | [2:03:27](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=7407s) |
| 08 | [Arrays & slices](08-arrays-and-slices.md) | Arrays, slice internals, len/cap, `make`, `append`, `range` | [2:22:34](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=8554s) |
| 09 | [Maps](09-maps.md) | Creating, mutating, comma-ok, maps vs structs | [3:19:30](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=11970s) |
| 10 | [Functions as values](10-functions-as-values.md) | Anonymous functions, higher-order functions, closures | [3:34:10](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=12850s) |
| 11 | [Methods](11-methods.md) | Receivers, value vs pointer receiver, when to use which | [3:42:21](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=13341s) |
| 12 | [Interfaces](12-interfaces.md) | Implicit satisfaction, empty interface, type assertion/switch, `Stringer`, `error`, DI & testing | [4:02:14](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=14534s) |
| 13 | [Generics](13-generics.md) | Type parameters, constraints, inference | [4:46:40](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=17200s) |
| 14 | [Concurrency](14-concurrency.md) | Goroutines, channels, buffering, `close`, `select`, mutexes | [4:53:57](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=17637s) |
| 15 | [Quick reference](15-quick-reference.md) | Syntax cheat sheet + all rules of thumb in one place | — |

---

## The short summary

Go puts code into **modules**. One module per project, described by a `go.mod` file. Each module contains **packages**. One package per directory. Package names are lowercase and short.

Capitalization controls visibility. A capitalized name is visible to other packages. A lowercase name is not.

Go is **statically typed**. It never converts types for you. You always write the conversion yourself.

Every variable starts with a **zero value**. Numbers start at `0`. Bools start at `false`. Strings start at `""`. Reference types start at `nil`. Nothing is ever undefined.

There are three data structures you will use constantly. **Structs** hold mixed fields and are for storing data. **Slices** are dynamic arrays and are the workhorse. **Maps** are for fast lookup.

You give a type behavior by attaching **methods** to it. You abstract over types with **interfaces**. Interfaces are satisfied automatically. A type just needs the right methods. This one idea powers `error`, `Stringer`, and the dependency injection pattern used in real Go backends.

Concurrency comes from **goroutines**. They are cheap and managed by Go, not the operating system. Goroutines talk to each other over **channels**. The guiding rule is: don't communicate by sharing memory, share memory by communicating.
