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

**When to use each:**

**Binary crate = "I want to run something"**
- You type `./my-program` or `cargo run` and it executes
- Examples:
  - Command-line tools (`grep`, `ls`, `cargo` itself)
  - Web servers (your API backend)
  - Desktop applications
  - Games
  - Scripts that do something

**Library crate = "I want to share reusable code"**
- You add it as a dependency in `Cargo.toml`
- Examples:
  - `serde` - serialization/deserialization
  - `tokio` - async runtime
  - `regex` - regular expressions
  - Your own shared utilities
  - Code that multiple projects need

**Real-world analogy:**
- **Binary** = a tool you pick up and use (hammer, calculator)
- **Library** = a component you build with (lumber, resistors)

You can have both! For example:
- `ripgrep` (the tool `rg`) is a binary
- But it's also published as a library so other Rust programs can use its search functionality

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

> **Why keep functions private? (Beyond separation of concerns)**
>
> **1. Freedom to change implementation**
> ```rust
> // Private helper - you can change it anytime
> fn calculate_internal(x: i32) -> i32 {
>     x * 2  // Later: change to x * 3, no one breaks
> }
>
> // Public API - changing this breaks other people's code
> pub fn calculate(x: i32) -> i32 {
>     calculate_internal(x)
> }
> ```
> Once something is `pub` in a library, changing it is a **breaking change**. Users depend on it. Private items can be refactored freely.
>
> **2. Prevent misuse**
> ```rust
> pub struct BankAccount {
>     balance: f64,  // Private! Can't be directly modified
> }
>
> impl BankAccount {
>     pub fn deposit(&mut self, amount: f64) {
>         if amount > 0.0 {  // Validation logic
>             self.balance += amount;
>         }
>     }
>     
>     // Without privacy, users could do: account.balance = -1000.0
> }
> ```
> Privacy enforces **invariants**—rules about valid states. Users must go through your validated public methods.
>
> **3. Reduce cognitive load**
> ```rust
> pub mod http_client {
>     pub fn get(url: &str) -> Response { }   // User sees this
>     pub fn post(url: &str) -> Response { }  // And this
>     
>     fn parse_headers() { }      // Hidden complexity
>     fn handle_redirects() { }   // Hidden complexity
>     fn retry_logic() { }        // Hidden complexity
> }
> ```
> Users only see 2 functions instead of 5. They don't need to know internal details.
>
> **4. API stability**
> - Public = promises you make to users
> - Private = implementation details
> - Fewer public items = easier to maintain without breaking changes
>
> **Rule of thumb:** Start private, make public only when needed. You can always expose more later, but you can't easily take back a public API.

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
> The `mod` keyword declares a **module**—a way to organize code into separate namespaces. You can define modules in two ways:
>
> **1. Inline (code in curly braces):**
> ```rust
> mod utils {
>     pub fn helper() {
>         println!("Helper function");
>     }
> }
> ```
>
> **2. External file (with semicolon):**
> ```rust
> mod utils;  // Tells compiler: look for utils.rs or utils/mod.rs
> ```
>
> When you use `mod utils;`, the compiler searches for the module code in this order:
> 1. `utils.rs` (a file next to the current file)
> 2. `utils/mod.rs` (older style, still supported)
>
> Think of `mod` as creating a tree structure. The crate root is the trunk, and each `mod` declaration adds a branch.

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

> **What do most Rust packages contain?**
>
> **Published to crates.io (the Rust package registry):**
> - **Library only** (most common) - Like `serde`, `tokio`, `regex`
> - **Library + binary** - Like `ripgrep` (library for code reuse, binary as a tool)
>
> **Your own projects:**
> - **Binary only** - Simple applications, one-off tools
> - **Library + binary** - Recommended pattern (library has logic, binary is thin wrapper)
>
> When you see "add this to your `Cargo.toml`" in documentation, you're adding a **library crate** as a dependency.

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

## Modules: Organizing Code Within a Crate

Modules let you organize code into namespaces and control privacy. Think of them like folders in a file system.

### Module Tree Structure

Every crate has a **module tree** starting from the implicit `crate` root. The tree is formed automatically based on your `mod` declarations—you don't create it explicitly.

**Example 1: Inline modules (all in one file)**

```rust
// src/lib.rs
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
        fn seat_at_table() {}
    }
    
    mod serving {
        fn take_order() {}
        fn serve_order() {}
    }
}
```

This automatically creates a tree:
```
crate
└── front_of_house
    ├── hosting
    │   ├── add_to_waitlist
    │   └── seat_at_table
    └── serving
        ├── take_order
        └── serve_order
```

**Example 2: Same tree, but split across files**

You get the **exact same tree** by writing `mod` separately in each file:

```rust
// src/lib.rs
mod front_of_house;  // Load from file
```

```rust
// src/front_of_house.rs
mod hosting;  // Load from file
mod serving;  // Load from file
```

```rust
// src/front_of_house/hosting.rs
fn add_to_waitlist() {}
fn seat_at_table() {}
```

```rust
// src/front_of_house/serving.rs
fn take_order() {}
fn serve_order() {}
```

**Same tree, different organization!** The compiler builds the tree by following the chain of `mod` declarations starting from the crate root.

> **Key insight:** You write `mod foo;` in each file where you want to declare a child module. The tree emerges from these declarations—it's not something you draw or configure separately.

**Terminology:**
- `hosting` and `serving` are **siblings** (both declared in `front_of_house.rs`)
- `hosting` is a **child** of `front_of_house` (declared inside it)
- `front_of_house` is the **parent** of `hosting` (contains its declaration)

> **How to visualize the module tree in your project**
>
> **1. Use `cargo modules` (requires cargo-modules)**
> ```bash
> cargo install cargo-modules
> cargo modules generate tree
> ```
> Output:
> ```
> crate my_project
> ├── mod front_of_house: pub
> │   ├── mod hosting: pub
> │   └── mod serving: pub
> └── fn eat_at_restaurant: pub
> ```
>
> **2. Use `tree` command on src/ directory**
> ```bash
> tree src/
> ```
> Shows file structure (which mirrors module structure):
> ```
> src/
> ├── lib.rs
> ├── front_of_house.rs
> └── front_of_house/
>     ├── hosting.rs
>     └── serving.rs
> ```
>
> **3. Generate documentation**
> ```bash
> cargo doc --open
> ```
> Cargo's documentation shows the full module hierarchy in the sidebar—great for understanding public API structure.
>
> **4. Quick manual check: grep for `mod` declarations**
> ```bash
> grep -r "^mod \|^pub mod " src/
> ```
> Shows all module declarations and where they are.
>
> **Pro tip:** File structure closely mirrors module tree for external modules, so `tree src/` is often enough!

### Module File Organization

You can organize modules across files. Here's a practical example:

```
my-restaurant/
├── Cargo.toml
└── src/
    ├── lib.rs              ← Crate root
    ├── front_of_house.rs   ← Module file
    └── front_of_house/     ← Submodules folder
        ├── hosting.rs
        └── serving.rs
```

```rust
// src/lib.rs
pub mod front_of_house;  // Loads src/front_of_house.rs

pub fn eat_at_restaurant() {
    front_of_house::hosting::add_to_waitlist();
}
```

```rust
// src/front_of_house.rs
pub mod hosting;  // Loads src/front_of_house/hosting.rs
pub mod serving;  // Loads src/front_of_house/serving.rs
```

```rust
// src/front_of_house/hosting.rs
pub fn add_to_waitlist() {
    println!("Added to waitlist");
}
```

**Where the compiler looks for `mod foo;`:**

| Declared in | Looks for code in |
|-------------|-------------------|
| `src/lib.rs` or `src/main.rs` | `src/foo.rs` or `src/foo/mod.rs` |
| `src/bar.rs` | `src/bar/foo.rs` or `src/bar/foo/mod.rs` |
| `src/bar/mod.rs` | `src/bar/foo.rs` or `src/bar/foo/mod.rs` |

> **Modern style:** Prefer `foo.rs` over `foo/mod.rs` unless you have submodules.

### Privacy Rules: The Crucial Part

Here's what trips people up—**privacy in modules is asymmetric**:

1. **Child can see parent's private items** ✓
2. **Parent cannot see child's private items** ✗
3. **Siblings cannot see each other's private items** ✗

```rust
mod parent {
    fn parent_fn() {}  // Private to module
    
    mod child {
        fn child_fn() {
            super::parent_fn();  // ✓ Child can access parent's private items
        }
    }
    
    fn another_fn() {
        child::child_fn();  // ✗ Error! Parent can't access child's private items
    }
}
```

**To fix it, make child items public:**

```rust
mod parent {
    fn parent_fn() {}
    
    pub mod child {          // Module itself is public
        pub fn child_fn() {  // Function inside is also public
            super::parent_fn();
        }
    }
    
    fn another_fn() {
        child::child_fn();  // ✓ Now it works!
    }
}
```

> **Key insight:** `pub mod` makes the *module* visible. You still need `pub` on items *inside* the module to make them usable.

### `pub mod` vs `mod`

```rust
// src/lib.rs
mod private_module {    // Only this crate can use it
    pub fn do_thing() {}
}

pub mod public_module {  // Other crates can use it
    pub fn do_thing() {}
}
```

```rust
// In another crate
use my_crate::public_module::do_thing;   // ✓ Works
use my_crate::private_module::do_thing;  // ✗ Error: module is private
```

**Rule of thumb:**
- Library crate? Use `pub mod` for modules you want to expose
- Binary crate? Usually just `mod` (no one else imports your code)

### Practical Example: A Full Module Setup

```
game/
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── player.rs
    └── enemies/
        ├── boss.rs
        └── minion.rs
```

```rust
// src/lib.rs
pub mod player;
pub mod enemies;

pub fn start_game() {
    player::spawn();
    enemies::spawn_wave();
}
```

```rust
// src/player.rs
pub fn spawn() {
    println!("Player spawned");
}

fn internal_state() {  // Private, only for this module
    // ...
}
```

```rust
// src/enemies.rs (note: this is enemies.rs, not enemies/mod.rs)
pub mod boss;
pub mod minion;

pub fn spawn_wave() {
    boss::spawn();
    minion::spawn_many(5);
}
```

```rust
// src/enemies/boss.rs
pub fn spawn() {
    println!("Boss spawned");
}
```

```rust
// src/enemies/minion.rs
pub fn spawn_many(count: u32) {
    for _ in 0..count {
        println!("Minion spawned");
    }
}
```

Now external code can do:
```rust
use game::player;
use game::enemies::boss;

player::spawn();
boss::spawn();
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
| **Module** | Namespace within a crate | `mod utils { }` or `mod utils;` |
| **Module tree** | Hierarchy of modules | Starts at `crate` root |

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

    Each crate contains:
    
    crate (root)
    ├── mod foo
    │   ├── pub fn bar()
    │   └── mod baz
    └── mod qux
        └── pub struct Thing
```

---

## Key Takeaways

1. **Package** = Cargo project with `Cargo.toml`
2. **Crate** = What the compiler compiles (binary or library)
3. **Module** = Namespace organization within a crate
4. **`pub`** = Makes things visible outside their module
5. **`mod`** = Declares a module (inline or external file)
6. **`use`** = Creates shortcuts to avoid long paths
7. **Privacy rule** = Child sees parent's privates, but parent can't see child's privates

