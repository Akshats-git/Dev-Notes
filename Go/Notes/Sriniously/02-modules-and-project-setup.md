# 02 — Modules & Project Setup

[▶ 27:51](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=1671s)

---

## The first program

```go
package main

import "fmt"

func main() {
	fmt.Println("hello 世界")
}
```

Run it:

```bash
go run main.go
```

Four things are happening here. A **package declaration**. An **import**. A **function named `main`**. And a call into the imported package.

---

## `go.mod`

[▶ 27:51](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=1671s)

```
module github.com/username/projectname

go 1.22
```

You create it with:

```bash
go mod init github.com/username/projectname
```

### What a `go.mod` file holds

It holds three things.

1. **The module name.** This is the name of your project. By convention you prefix it with your code hosting provider. That means `github.com/`, `gitlab.com/`, or `bitbucket.org/`, followed by your username or organization.
2. **The Go version.** This is the *minimum* Go version the project needs. It is not necessarily the version installed on your machine.
3. **External dependencies.** Every package the project uses, pinned to an exact version.

### Why the hosting provider prefix

Because a Go project is one of two kinds.

| Kind | What it is | What you do with it |
|---|---|---|
| **Executable** | A program you run | You **run** it |
| **Library / package** | Code others import | You **import** it |

Backend projects are executables.

But the module path is also the import path if anyone ever imports your code. So the unique URL-shaped name is the convention either way.

### Working with dependencies

```bash
go get github.com/aws/aws-sdk-go   # adds the package + version to go.mod
go mod download                     # a teammate fetches every pinned dep in one command
```

That second command is the payoff. A teammate clones your repo and runs one command. They get the exact same dependency versions you have. They do not have to run `go get` once per package.

**You will almost never edit `go.mod` by hand.** The commands maintain it for you. The one exception is pinning a specific version manually.

### Real-world look

fider's `go.mod` has the same three parts. A module name of `github.com/getfider/fider`. A Go version line. Then a long `require` block listing external packages with exact versions. Same structure, just bigger.

---

## `main.go` explained

[▶ 37:13](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=2233s)

### The `main` package rule

Every `.go` file starts with a package declaration.

For an **executable**, three rules apply:

- There must be a package named `main`.
- Inside it there must be a `func main()`.
- That function is the **entry point**. Execution starts there.

### Imports

```go
import "fmt"
```

There is no prefix here. That means it comes from the **standard library**.

Standard library packages ship with Go. You do not install them. But you still have to import them.

A prefix like `github.com/...` means it is an external package, or a package from your own module.

`fmt` handles formatting, printing, and string manipulation.

### The capital letter

```go
fmt.Println(...)   // capital P — not optional
```

Anything you access from *another* package must start with a capital letter. This is how Go does encapsulation. It is covered in [03 — Packages & visibility](03-packages-and-visibility.md).

---

## UTF-8 support

[▶ 44:43](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=2683s)

The `世界` in the hello world string is not decoration. It shows that Go source code and Go strings are **UTF-8** by default.

Older languages relied on **ASCII**. ASCII has 128 code points (0 to 127). That covers a–z, A–Z, 0–9, and some punctuation. It is very limiting. The world has thousands of writing systems.

Go was built on **Unicode**, encoded as **UTF-8**. That is roughly 150,000 characters across about 160 scripts. It covers essentially every written language.

Two related types (see [05 — Variables & types](05-variables-and-types.md)):

- `byte` is an alias for `uint8`. It represents the raw bytes of a string.
- `rune` is an alias for `int32`. It represents one Unicode character.

---

## `go build` vs `go run`

[▶ 47:00](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=2820s)

```bash
go build main.go   # produces an executable file named "main"
./main             # run it
```

```bash
go run main.go     # compile and execute in one step, no binary left behind
```

### Cross-compilation

This is one of Go's best features.

```bash
GOOS=windows GOARCH=amd64 go build .   # produces a .exe, from macOS or Linux
GOOS=linux   GOARCH=amd64 go build .
```

One command produces a **single self-contained binary** for a different operating system and architecture.

The Go runtime and all dependencies get embedded into that binary. You can ship it to a machine that has no Go installed. It just runs.

If you leave out `GOOS` and `GOARCH`, it builds for your own platform.

This is why Go is easy to deploy. There is no runtime to install on the server. There is no dependency tree to resolve there either.

### Which one to use

| Command | Use it | What it does |
|---|---|---|
| `go run` | **Local development** | Compiles, executes from a temp location, leaves no artifact |
| `go build` | **Deployment, CI/CD, distribution** | Compiles and writes a binary you can ship |

The only real difference is where the binary ends up. `go build` writes it into your directory. `go run` builds it into a temp directory, runs it, and throws it away.

Both use the same build cache (`go env GOCACHE`). So both get fast incremental rebuilds. Neither is meaningfully faster than the other.

---

**Next:** [03 — Packages & visibility](03-packages-and-visibility.md)
