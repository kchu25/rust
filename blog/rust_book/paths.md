@def title = "Rust Paths: Navigating the Module Tree"
@def published = "30 January 2026"
@def tags = ["rust-book-ch7-modules"]

# Rust Paths: Navigating the Module Tree

Once you have a module tree, you need a way to refer to items in it. That's where **paths** come in—they're like file system paths, but for your code.

---

## What is a Path?

A **path** tells Rust where to find an item (function, struct, enum, etc.) in your module tree.

**Simply put:** A path is `x::y::z` where `::` separates each level of the module hierarchy.

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Path to the function - NO `use` needed!
    front_of_house::hosting::add_to_waitlist();
    //     ↓          ↓              ↓
    //   module   submodule      function
}
```

Each `::` navigates deeper into the tree, just like `/` in a file path (`home/user/file.txt`).

> **`mod` vs `use` - What's the difference?**
>
> - **`mod front_of_house;`** - Declares/includes the module in your tree
> - **`use front_of_house::hosting;`** - Creates a shortcut (optional convenience)
>
> If you've declared a module with `mod`, you can **always** access it with a path—no `use` required!
>
> ```rust
> mod utils {
>     pub fn helper() {}
> }
>
> fn main() {
>     utils::helper();           // ✓ Works! No `use` needed
>     
>     // You CAN use `use` for convenience:
>     use utils::helper;
>     helper();                  // ✓ Also works, shorter
> }
> ```
>
> **`use` is purely for convenience**—it creates shortcuts to avoid repeating long paths.

**Think of it like a file path:**
- File system: `/home/user/documents/file.txt`
- Module path: `crate::front_of_house::hosting::add_to_waitlist`

The `::` separator is like `/` in a file path—it navigates through the module hierarchy.

---

## Two Types of Paths

### 1. Absolute Path (starts from crate root)

**Always starts with `crate`** (which means the root of your crate, not your current location)

```rust
crate::front_of_house::hosting::add_to_waitlist();
```

> **What is the crate root?**
>
> The **crate root** is the file where your crate's module tree begins:
>
> | Crate Type | Root File |
> |------------|-----------|
> | **Binary crate** | `src/main.rs` |
> | **Library crate** | `src/lib.rs` |
>
> Everything you declare in these files is at the root level. When you use `crate::`, you're referring to items declared in these root files (or modules declared there).
>
> ```rust
> // src/lib.rs (this IS the crate root)
> pub fn root_function() {}    // At the root
> 
> mod foo {                    // At the root
>     pub fn bar() {}
> }
>
> // Anywhere in your crate:
> crate::root_function();      // ✓ Accesses the root
> crate::foo::bar();           // ✓ Goes: root → foo → bar
> ```
>
> **Multiple binaries?** Each has its own root:
> - `src/main.rs` is the root for the main binary
> - `src/bin/tool.rs` is the root for the `tool` binary
> - `src/lib.rs` is the root for the library

> **Important:** `crate::` **always** starts from the very top of your crate (the root), no matter where you are in the module tree.
>
> ```rust
> // In src/lib.rs or src/main.rs (the root)
> mod foo {
>     pub mod bar {
>         pub fn baz() {
>             // `crate::` goes back to the ROOT, not to `bar` or `foo`
>             crate::foo::bar::baz();  // Goes: root → foo → bar → baz
>         }
>     }
> }
> ```
>
> Think of `crate` as the root `/` in a file system—it's an **absolute** starting point, not relative to where you are.

Like an absolute file path: `/home/user/file.txt` (the `/` always means root)

**When to use:**
- When the item and caller might move independently
- When you want clarity about where something lives
- Generally preferred for stability

### 2. Relative Path (starts from current module)

**Starts from where you are** in the module tree

```rust
front_of_house::hosting::add_to_waitlist();
```

Like a relative file path: `../documents/file.txt`

**When to use:**
- When items will likely move together
- When you want shorter paths
- Inside tightly coupled modules

---

## Absolute vs Relative: A Comparison

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path - starts from crate root
    crate::front_of_house::hosting::add_to_waitlist();
    
    // Relative path - starts from current location
    front_of_house::hosting::add_to_waitlist();
}
```

Both work! The difference matters when you refactor:

**Scenario: Move `eat_at_restaurant` to a submodule**

```rust
mod dining {
    pub fn eat_at_restaurant() {
        // Absolute path still works!
        crate::front_of_house::hosting::add_to_waitlist();
        
        // Relative path breaks - front_of_house isn't a sibling anymore
        // front_of_house::hosting::add_to_waitlist(); // ✗ Error!
    }
}
```

**Scenario: Move `front_of_house` and `eat_at_restaurant` together**

```rust
mod customer_experience {
    mod front_of_house {
        pub mod hosting {
            pub fn add_to_waitlist() {}
        }
    }
    
    pub fn eat_at_restaurant() {
        // Absolute path needs updating
        crate::customer_experience::front_of_house::hosting::add_to_waitlist();
        
        // Relative path still works!
        front_of_house::hosting::add_to_waitlist();
    }
}
```

---

## Using `super` for Parent Modules

`super` is like `..` in file systems—it goes to the parent module.

```rust
fn deliver_order() {}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();              // Sibling in same module
        super::deliver_order();    // Parent's function
    }
    
    fn cook_order() {}
}
```

**Module tree:**
```
crate
├── deliver_order()
└── back_of_house
    ├── fix_incorrect_order()
    └── cook_order()
```

`super` from `back_of_house` goes up to `crate`, then finds `deliver_order`.

### When to use `super`

```rust
mod restaurant {
    fn notify_manager() {
        println!("Notifying manager");
    }
    
    mod kitchen {
        pub fn report_issue() {
            super::notify_manager();  // ✓ Clear: go to parent
            crate::restaurant::notify_manager(); // ✗ Verbose
        }
    }
}
```

**Use `super` when:**
- Accessing parent module items
- You expect the child and parent to stay together
- You want concise, relative navigation

---

## Using `self` for Current Module

`self` refers to the current module (rarely needed, but useful for clarity).

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
        
        pub fn seat_at_table() {
            self::add_to_waitlist();  // Explicit: call sibling
            add_to_waitlist();         // Implicit: same thing
        }
    }
}
```

Most people omit `self` since it's implicit, but it can make intent clearer in complex code.

---

## Privacy and Paths

**Critical rule:** Every segment in a path must be `pub` for external access.

```rust
mod front_of_house {
    pub mod hosting {          // ✓ Module is public
        pub fn add_to_waitlist() {}  // ✓ Function is public
    }
    
    mod private_room {         // ✗ Module is private
        pub fn reserve() {}    // Function is pub, but can't reach it!
    }
}

pub fn eat_at_restaurant() {
    front_of_house::hosting::add_to_waitlist();  // ✓ Works
    front_of_house::private_room::reserve();      // ✗ Error: module private
}
```

### Making Paths Accessible

You need `pub` at **every level**:

```rust
pub mod level1 {           // Must be pub
    pub mod level2 {       // Must be pub
        pub fn my_fn() {}  // Must be pub
    }
}

// Now accessible from outside:
use crate::level1::level2::my_fn;
```

**Common mistake:** Making a module `pub` but forgetting its contents

```rust
mod restaurant {
    pub mod kitchen {      // Module is public
        fn cook() {}       // ✗ Function is still private!
    }
}

// Error: function `cook` is private
// restaurant::kitchen::cook();
```

---

## Structs and Enums: Special Cases

### Structs: Opt-in Public Fields

```rust
mod back_of_house {
    pub struct Breakfast {
        pub toast: String,        // Public field
        seasonal_fruit: String,   // Private field
    }
    
    impl Breakfast {
        pub fn summer(toast: &str) -> Breakfast {
            Breakfast {
                toast: String::from(toast),
                seasonal_fruit: String::from("peaches"),
            }
        }
    }
}

pub fn eat_at_restaurant() {
    let mut meal = back_of_house::Breakfast::summer("Rye");
    meal.toast = String::from("Wheat");     // ✓ Public field
    // meal.seasonal_fruit = "berries";     // ✗ Private field
}
```

**Key insight:** If a struct has private fields, you **must** provide a constructor (like `summer()`) because external code can't set private fields directly.

### Enums: All-or-Nothing Public

```rust
mod back_of_house {
    pub enum Appetizer {
        Soup,    // Automatically public
        Salad,   // Automatically public
    }
}

pub fn eat_at_restaurant() {
    let order1 = back_of_house::Appetizer::Soup;   // ✓ Works
    let order2 = back_of_house::Appetizer::Salad;  // ✓ Works
}
```

**Why the difference?**
- **Structs** often have internal state you want to protect (invariants)
- **Enums** are useless if you can't access their variants

---

## Practical Examples

### Example 1: Restaurant Module

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
        pub fn seat_at_table() {}
    }
    
    pub mod serving {
        pub fn take_order() {}
        pub fn serve_order() {}
        pub fn take_payment() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute paths
    crate::front_of_house::hosting::add_to_waitlist();
    crate::front_of_house::serving::take_order();
    
    // Relative paths (shorter)
    front_of_house::hosting::seat_at_table();
    front_of_house::serving::serve_order();
}
```

### Example 2: Using `super` in Nested Modules

```rust
mod sound {
    pub fn guitar() {
        println!("🎸");
    }
    
    pub mod instrument {
        pub fn clarinet() {
            println!("🎵");
            super::guitar();  // Access parent's function
        }
    }
}

pub fn main() {
    sound::instrument::clarinet();
    // Output:
    // 🎵
    // 🎸
}
```

### Example 3: Sibling Access with Privacy

```rust
mod performance {
    pub mod stage {
        pub fn perform() {
            // Can access sibling through parent
            super::backstage::prepare();  // ✓ Works
        }
    }
    
    mod backstage {  // Private module
        pub fn prepare() {
            println!("Getting ready...");
        }
    }
}

// From outside the parent module:
// performance::backstage::prepare();  // ✗ Error: backstage is private
```

**Key insight:** Siblings can access each other through their common parent, even if one is private—as long as you're calling from within the parent module.

---

## Common Patterns

### Pattern 1: Library with Binary

```rust
// src/lib.rs
pub mod config {
    pub fn load() -> Config { /* ... */ }
}

pub mod parser {
    pub fn parse(input: &str) -> Result { /* ... */ }
}
```

```rust
// src/main.rs
use my_app::config;
use my_app::parser;

fn main() {
    let cfg = config::load();
    let result = parser::parse("input");
}
```

**Best practice:** Define your module tree in `lib.rs`, use it from `main.rs`. The binary becomes a thin client of your library.

### Pattern 2: Nested Module Organization

```rust
// src/lib.rs
pub mod network {
    pub mod server {
        pub fn start() {}
    }
    
    pub mod client {
        pub fn connect() {
            // Access sibling through parent
            super::server::start();
        }
    }
}
```

### Pattern 3: Re-exporting for Convenience

```rust
mod internal {
    pub mod deeply {
        pub mod nested {
            pub fn useful_fn() {}
        }
    }
}

// Re-export for shorter paths
pub use internal::deeply::nested::useful_fn;

// Users can now do:
// use my_crate::useful_fn;
// Instead of:
// use my_crate::internal::deeply::nested::useful_fn;
```

---

## Path Resolution Rules

**How Rust finds items:**

1. **Absolute path** (`crate::...`): Start at crate root
2. **Relative path** (module name): Start at current module
3. **`super::`**: Go to parent module
4. **`self::`**: Stay in current module (usually implicit)

**Privacy checks at each step:**
- Is the module `pub`?
- Is the item inside `pub`?
- Am I in the right scope to access private items?

---

## Quick Reference

| Path Type | Syntax | Example | When to Use |
|-----------|--------|---------|-------------|
| **Absolute** | `crate::...` | `crate::foo::bar()` | Default choice, stable |
| **Relative** | `module::...` | `foo::bar()` | Tightly coupled code |
| **Parent** | `super::...` | `super::parent_fn()` | Access parent module |
| **Current** | `self::...` | `self::sibling()` | Explicit current module |

---

## Common Errors and Solutions

### Error: "module is private"

```rust
mod private {
    pub fn func() {}
}

// Error: module `private` is private
// use private::func;
```

**Solution:** Make the module `pub`:
```rust
pub mod private {
    pub fn func() {}
}
```

### Error: "function is private"

```rust
pub mod public_mod {
    fn private_fn() {}
}

// Error: function `private_fn` is private
// use public_mod::private_fn;
```

**Solution:** Make the function `pub`:
```rust
pub mod public_mod {
    pub fn private_fn() {}
}
```

### Error: Path doesn't exist

```rust
// Error: cannot find `foo` in this scope
foo::bar();
```

**Solutions:**
- Add `mod foo;` to declare the module
- Use correct path: `crate::foo::bar()` or `super::foo::bar()`
- Import with `use`: `use crate::foo;`

---

## Key Takeaways

1. **Paths navigate the module tree** - Like file system paths, but for code
2. **Absolute paths** (`crate::`) - Start from root, stable when refactoring
3. **Relative paths** - Start from current module, shorter but fragile
4. **`super`** - Navigate to parent module
5. **Every segment must be `pub`** - For external access
6. **Structs vs Enums** - Structs have private fields by default, enum variants are always public
7. **Use absolute paths by default** - Unless you have a reason to use relative

**Next up:** The `use` keyword for creating shortcuts to long paths!
