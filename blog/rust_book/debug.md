@def title = "Understanding #[derive(Debug)] in Rust"
@def published = "30 January 2026"
@def tags = ["rust-basics"]

# Understanding #[derive(Debug)] in Rust

You'll often see `#[derive(Debug)]` above structs and enums in Rust code. Let's understand what it does and why it's so common.

---

## What is `#[derive(Debug)]`?

```rust
#[derive(Debug)]
pub struct Player {
    name: String,
    health: u32,
}
```

**What it does:** Automatically implements the `Debug` trait, which lets you print the struct for debugging purposes.

---

## Why You Need It

**With `Debug`:**
```rust
#[derive(Debug)]
pub struct Player {
    name: String,
    health: u32,
}

let player = Player { 
    name: "Alice".to_string(), 
    health: 100 
};

println!("{:?}", player);  
// Output: Player { name: "Alice", health: 100 }
```

**Without `Debug`:**
```rust
pub struct Player {
    name: String,
    health: u32,
}

let player = Player { 
    name: "Alice".to_string(), 
    health: 100 
};

println!("{:?}", player);  
// ✗ Error: Player doesn't implement Debug
```

---

## When You Need `Debug`

### 1. Using `{:?}` format specifier

> **What is `{:?}`?**
>
> `{:?}` is a **format specifier** used in formatting macros like `println!`, `format!`, etc. It tells Rust to use the `Debug` trait to format the value.
>
> Think of it like format codes in other languages:
> - `{}` = use `Display` trait (user-friendly output)
> - `{:?}` = use `Debug` trait (developer-friendly output)
> - `{:#?}` = use `Debug` trait with pretty-printing (multi-line, indented)
>
> ```rust
> let name = "Alice";
> let age = 30;
> 
> println!("Name: {}", name);        // Display: Name: Alice
> println!("Age: {:?}", age);        // Debug: Age: 30
> println!("Data: {:#?}", vec![1,2,3]); // Pretty Debug (multi-line)
> ```

```rust
#[derive(Debug)]
struct Point { x: i32, y: i32 }

let p = Point { x: 10, y: 20 };
println!("{:?}", p);       // Point { x: 10, y: 20 }
println!("{:#?}", p);      // Pretty-print (multi-line)
// Output with {:#?}:
// Point {
//     x: 10,
//     y: 20,
// }
```

### 2. Using the `dbg!()` macro

```rust
#[derive(Debug)]
struct Config { port: u16, host: String }

let config = Config { 
    port: 8080, 
    host: "localhost".to_string() 
};

dbg!(&config);  // Prints to stderr with file/line info
// [src/main.rs:10] &config = Config { port: 8080, host: "localhost" }
```

### 3. During development and testing

```rust
#[derive(Debug)]
struct User { id: u32, name: String }

#[test]
fn test_user_creation() {
    let user = User { id: 1, name: "Bob".to_string() };
    println!("Created user: {:?}", user);  // Helpful for debugging tests
    assert_eq!(user.id, 1);
}
```

---

## `Debug` vs `Display`

```rust
use std::fmt;

#[derive(Debug)]
struct Person {
    name: String,
    age: u32,
}

impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{} (age {})", self.name, self.age)
    }
}

let person = Person { name: "Alice".to_string(), age: 30 };

println!("{:?}", person);   // Debug: Person { name: "Alice", age: 30 }
println!("{}", person);     // Display: Alice (age 30)
```

| Trait | Format | Purpose | Usage |
|-------|--------|---------|-------|
| **Debug** | `{:?}` or `{:#?}` | For developers (debugging) | Can be auto-derived |
| **Display** | `{}` | For end users (readable output) | Must implement manually |

**Rule of thumb:** 
- `Debug` = what developers need to see
- `Display` = what users need to see

---

## Other Common Derives

`Debug` is rarely used alone. Here are common combinations:

```rust
#[derive(Debug, Clone)]
struct Document {
    content: String,
}
// Can debug AND clone

#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}
// Can debug AND compare for equality

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct UserId(u64);
// Common pattern for ID types - can do everything!

#[derive(Debug, Default)]
struct Config {
    port: u16,
    host: String,
}
// Can debug AND create with Config::default()
```

### Most Common Derive Combinations

| Derives | Use Case |
|---------|----------|
| `Debug` | Minimum for development |
| `Debug, Clone` | Need to duplicate values |
| `Debug, PartialEq` | Need to compare values |
| `Debug, Clone, PartialEq` | Most common general-purpose |
| `Debug, Clone, Copy` | Small types (no heap data) |
| `Debug, Default` | Types with sensible default values |

---

## How `derive` Works

The `derive` attribute **auto-generates** trait implementations that you'd otherwise write manually.

**Without derive (manual implementation):**
```rust
struct Point { x: i32, y: i32 }

impl std::fmt::Debug for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
```

**With derive (automatic):**
```rust
#[derive(Debug)]
struct Point { x: i32, y: i32 }
```

Same result, much less code!

---

## When NOT to Derive Debug

Sometimes you want **custom** debug output:

```rust
struct Password(String);

// Don't derive Debug - it would expose the password!
impl std::fmt::Debug for Password {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Password([REDACTED])")
    }
}

let pw = Password("secret123".to_string());
println!("{:?}", pw);  // Password([REDACTED])
```

Or when debugging large structs, you might want abbreviated output:

```rust
struct LargeDataset {
    data: Vec<f64>,  // Could be millions of elements
    name: String,
}

impl std::fmt::Debug for LargeDataset {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "LargeDataset {{ name: {:?}, size: {} }}", 
               self.name, self.data.len())
    }
}
```

---

## Derivable Traits

Not all traits can be derived. Here are the standard library traits that support `derive`:

### Commonly Used
- `Debug` - Debug formatting
- `Clone` - Explicit duplication
- `Copy` - Implicit duplication (for simple types)
- `PartialEq` - Equality comparison
- `Eq` - Full equality (requires `PartialEq`)
- `PartialOrd` - Partial ordering
- `Ord` - Total ordering (requires `PartialOrd`, `Eq`)
- `Hash` - Hashing support
- `Default` - Default value construction

### Less Common
- `From` - Type conversion (with `#[derive(From)]` from external crates)
- Many more with external crates like `serde`'s `Serialize`/`Deserialize`

---

## Key Takeaways

1. **`#[derive(Debug)]` is essential** - Almost every struct/enum should have it
2. **`Debug` is for developers** - Use `Display` for user-facing output
3. **`derive` saves time** - Auto-generates trait implementations
4. **Start with `Debug`** - Add more derives as needed
5. **Custom implementations** - Override when you need special behavior (like hiding sensitive data)

**Rule of thumb:** When in doubt, add `Debug`. You can always customize it later if needed.
