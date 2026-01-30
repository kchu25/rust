@def title = "Rust Packages and Crates: Untangled"
@def published = "30 January 2026"
@def tags = ["rust-basics"]

# Rust Packages and Crates: Untangled

The official Rust book introduces three concepts at once—crates, packages, and modules—which can feel overwhelming. Let's untangle them one layer at a time.

---

## The Big Picture

```
Package (has Cargo.toml)
├── Crate (binary) ← compiled to executable
├── Crate (library) ← compiled to shareable code
└── Crate (binary) ← you can have multiple binaries
```

Think of it like this:
- **Crate** = a unit of compilation (what `rustc` compiles)
- **Package** = a bundle of crates managed by Cargo (has `Cargo.toml`)
- **Module** = organization within a crate (we'll cover this separately)

---

## What is a Crate?

A **crate** is the smallest unit of code the Rust compiler deals with. When you run `rustc` on a file, that file is a crate.

### Two Kinds of Crates

| Type | Has `main`? | Compiles to | Example |
|------|-------------|-------------|---------|
| **Binary crate** | Yes | Executable you can run | CLI tool, server, game |
| **Library crate** | No | Code others can use | `rand`, `serde`, `tokio` |

```rust
// Binary crate - has main(), runs as a program
fn main() {
    println!("I'm an executable!");
}

// Library crate - no main(), provides functionality
pub fn useful_function() {
    // Other code can call this
}
```

> **What does `pub` mean?**
>
> The `pub` keyword means **public**. By default, everything in Rust is *private*—only accessible within the same module. Adding `pub` makes an item visible to code outside its module.
>
> ```rust
> fn private_function() { }      // Only usable within this module
> pub fn public_function() { }   // Usable by external code
> ```
>
> For a library crate, you *must* mark functions as `pub` if you want other code to use them. Without `pub`, your library would be useless—nothing would be accessible!

> **How `pub` relates to `use`**
>
> The `use` keyword brings items into scope so you don't have to write full paths. But you can only `use` items that are **public** (`pub`) to you:
>
> ```rust
> // In some library crate "mylib"
> pub fn public_fn() { }     // ✓ Can be imported
> fn private_fn() { }        // ✗ Cannot be imported from outside
>
> pub mod utils {            // Module is public
>     pub fn helper() { }    // ✓ Can be imported
>     fn secret() { }        // ✗ Cannot be imported from outside
> }
> ```
>
> ```rust
> // In your code
> use mylib::public_fn;           // ✓ Works
> use mylib::private_fn;          // ✗ Error: private
> use mylib::utils::helper;       // ✓ Works (both mod and fn are pub)
> use mylib::utils::secret;       // ✗ Error: private
> ```
>
> The `::` in paths like `mylib::utils::helper` is just navigating through modules—like folders in a file path. Each segment must be `pub` for external code to reach through it.

> **What can `use` import?**
>
> `use` can import almost anything—not just functions:
>
> ```rust
> use std::collections::HashMap;  // struct
> use std::io::Result;            // type alias
> use std::cmp::Ordering;         // enum
> use std::fmt::Debug;            // trait
> use std::f64::consts::PI;       // constant
> use mylib::utils;               // module itself
> use mylib::utils::helper;       // function
> ```
>
> When you import a **module**, you can then access its contents with `::`:
> ```rust
> use std::collections;           // import the module
> let map = collections::HashMap::new();  // use items inside it
> ```
>
> When you import a **specific item**, you use it directly:
> ```rust
> use std::collections::HashMap;  // import the struct directly
> let map = HashMap::new();       // no prefix needed
> ```
>
> **TL;DR:** `use` controls how much of the namespace path you want visible. You're choosing where to "stop" in the path hierarchy.

> **"Crate" usually means library**
> 
> When Rustaceans say "I'm using the `serde` crate," they mean library crate. It's interchangeable with "library" in casual conversation.

### The Crate Root

Every crate has a **root file**—the starting point for compilation:

| Crate type | Root file |
|------------|-----------|
| Binary | `src/main.rs` |
| Library | `src/lib.rs` |

The compiler starts at the root and follows all the `mod` declarations to find the rest of your code.

> **What does `mod` mean?**
>
> The `mod` keyword declares a **module**—a way to organize code into separate namespaces. When the compiler sees `mod foo;`, it looks for the code in either:
> - `foo.rs` (a file), or
> - `foo/mod.rs` (a folder with a mod.rs file)
>
> ```rust
> // In src/lib.rs or src/main.rs
> mod utils;    // Tells compiler: "look for utils.rs or utils/mod.rs"
> mod helpers;  // Tells compiler: "look for helpers.rs or helpers/mod.rs"
> ```
>
> Think of `mod` as creating a tree structure. The crate root is the trunk, and each `mod` declaration adds a branch. We'll cover modules in detail in a separate post.

---

## What is a Package?

A **package** is what you create when you run `cargo new`. It's a directory with:
- A `Cargo.toml` file (the manifest)
- One or more crates

> **What's a manifest?**
>
> A **manifest** is a file that describes your project's metadata and configuration. In Rust, it's `Cargo.toml`. It tells Cargo:
> - What your package is called
> - What version it is
> - What dependencies it needs
> - How to build it
>
> Think of it like a shipping label + packing list for your code. The name comes from shipping terminology—a ship's manifest lists everything on board.

```bash
$ cargo new my-project
     Created binary (application) `my-project` package

$ tree my-project
my-project
├── Cargo.toml    ← Package manifest
└── src
    └── main.rs   ← Binary crate root
```

### Package Rules

1. **At least one crate** (binary or library)
2. **At most one library crate** (you can only have one `src/lib.rs`)
3. **Any number of binary crates** (multiple executables are fine)

---

## Package Configurations

### Binary Only (most common for applications)

```
my-app/
├── Cargo.toml
└── src/
    └── main.rs      ← Binary crate "my-app"
```

This is what `cargo new my-app` creates.

### Library Only (for sharing code)

```
my-lib/
├── Cargo.toml
└── src/
    └── lib.rs       ← Library crate "my-lib"
```

This is what `cargo new my-lib --lib` creates.

### Both Binary and Library

```
my-project/
├── Cargo.toml
└── src/
    ├── main.rs      ← Binary crate "my-project"
    └── lib.rs       ← Library crate "my-project"
```

The binary can use the library:

```rust
// src/main.rs
use my_project::useful_function;  // Import from the library crate

fn main() {
    useful_function();
}
```

```rust
// src/lib.rs
pub fn useful_function() {
    println!("Called from the library!");
}
```

### Multiple Binaries

```
my-project/
├── Cargo.toml
└── src/
    ├── main.rs          ← Binary "my-project"
    ├── lib.rs           ← Library "my-project"
    └── bin/
        ├── tool1.rs     ← Binary "tool1"
        └── tool2.rs     ← Binary "tool2"
```

Run them with:
```bash
cargo run --bin my-project
cargo run --bin tool1
cargo run --bin tool2
```

---

## Convention Over Configuration

Cargo uses **file location** to determine crate structure. You don't need to specify these in `Cargo.toml`:

| File exists | Cargo assumes |
|-------------|---------------|
| `src/main.rs` | Binary crate with package name |
| `src/lib.rs` | Library crate with package name |
| `src/bin/foo.rs` | Additional binary crate named "foo" |

This is why `Cargo.toml` often has no explicit crate configuration—Cargo just looks at your file structure.

---

## Why Separate Binary and Library?

A common pattern: put your logic in a library crate, and keep `main.rs` thin.

**Benefits:**
- Library can be tested independently
- Library can be used by other projects
- Binary is just a thin wrapper that calls library functions

```rust
// src/lib.rs - All the real logic
pub fn run_app(args: Vec<String>) -> Result<(), String> {
    // Complex logic here
    Ok(())
}

// src/main.rs - Just the entry point
fn main() {
    let args: Vec<String> = std::env::args().collect();
    if let Err(e) = my_project::run_app(args) {
        eprintln!("Error: {}", e);
        std::process::exit(1);
    }
}
```

---

## Quick Reference

| Term | What it is | Example |
|------|------------|---------|
| **Crate** | Compilation unit | `rand`, your `main.rs` |
| **Binary crate** | Compiles to executable | CLI app, server |
| **Library crate** | Compiles to reusable code | `serde`, `tokio` |
| **Crate root** | Entry point for compiler | `src/main.rs`, `src/lib.rs` |
| **Package** | Bundle of crates + `Cargo.toml` | What `cargo new` creates |

---

## Mental Model

```
┌─────────────────────────────────────────┐
│  Package (Cargo.toml)                   │
│  "A project managed by Cargo"           │
│                                         │
│  ┌──────────────┐  ┌──────────────┐    │
│  │ Binary Crate │  │ Library Crate│    │
│  │ (main.rs)    │  │ (lib.rs)     │    │
│  │              │  │              │    │
│  │ → Executable │  │ → .rlib file │    │
│  └──────────────┘  └──────────────┘    │
│                                         │
│  ┌──────────────┐  ┌──────────────┐    │
│  │ Binary Crate │  │ Binary Crate │    │
│  │ (bin/a.rs)   │  │ (bin/b.rs)   │    │
│  └──────────────┘  └──────────────┘    │
└─────────────────────────────────────────┘
```

**Next up:** Modules—how to organize code *within* a crate.
