# 03 — Packages & Visibility

[▶ 51:48](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=3108s)

---

## Why packages exist

You can write everything in one file. It works fine for a small script.

But programs grow. At thousands of lines, then millions, a single file becomes impossible to read and impossible to debug.

Every language solves this somehow. JavaScript has CommonJS and ESM imports and exports. Go has exactly two levels.

1. **Modules.** One per project. You create it once with `go mod init` and then forget about it.
2. **Packages.** This is the real code organization unit.

That is the entire story. Go has nothing else for organizing code.

---

## The rules

- **Every `.go` file** starts with a package declaration. The filename does not matter.
- **One package per directory.** Every file in a folder declares the same package.
- **By convention the package name matches the folder name.**
- Test files conventionally add a `_test` suffix to the package name.
- An **executable** must have `package main` containing `func main()`.

```go
package handlers   // file lives in the handlers/ directory
```

---

## Import paths: only the last part matters

```go
import "math/rand"

func main() {
	fmt.Println(rand.Intn(10))   // referred to as "rand", not "math/rand"
}
```

The prefix is only there so Go can find the package. **The last segment is the name you actually type in your code.**

```go
import (
	"crypto"
	"crypto/rsa"
	"crypto/sha1"
	"fmt"
	"net/http"
)
```

When you have several imports, put them in a parenthesized block. One per line.

---

## Package naming conventions

There are two rules. Both exist because you type the package name at every use site.

1. **All lowercase.** Like `main`, `fmt`, `rand`, `handlers`, `actions`. Never capitalized.
2. **As short as possible.** This is a strong convention rather than a hard rule. You will write this name hundreds of times. A ten-letter package name is a tax on every line.

### Real-world check (fider)

| Folder | Package | Notes |
|---|---|---|
| `actions/` | `actions` | matches folder, short, lowercase |
| `handlers/` | `handlers` | every file in the folder declares it |
| `webhooks/` | `webhooks` | same pattern |

Test files in `actions/` use `actions_test`.

---

## Visibility: capitalization is the access modifier

[▶ 57:20 area](https://www.youtube.com/watch?v=tgGNwG_UxFo&t=3440s)

This is how Go does encapsulation. It applies to **everything**. Variables, constants, functions, types, struct fields, methods, and interfaces.

| First letter | Visibility |
|---|---|
| **Capital** | **Exported.** Other packages can access it. |
| lowercase | **Unexported.** Only its own package can access it. |

```go
import "math"

func main() {
	fmt.Println(math.pi)  // ❌ undefined: math.pi
	fmt.Println(math.Pi)  // ✅
}
```

You get this error at **compile time**.

And if your editor has Go LSP support, you see it as you type. That works in VS Code with the Go extension, in GoLand, and in Neovim with gopls. That instant feedback is a real reason to use an editor with proper Go support.

### Why it matters in practice

If a struct field is lowercase, other packages cannot read it or write it.

The video shows this in fider. He lowercases one exported field of a struct. Errors immediately appear in every file in other packages that used it.

If you come from an object-oriented language, this maps onto `public` and `private`. But do not try to think in objects when writing Go. It is the same *idea* (hide the implementation, expose an interface). It is not the same model.

---

**Next:** [04 — Functions](04-functions.md)
