# Mastering Pinning in Rust

> **Source:** https://freedium.cfd/https://blog.devgenius.io/mastering-pinning-in-rust-89c29f4e5567

Pinning in Rust is an essential concept for scenarios where certain values in memory must remain in a fixed location, making it critical…

## Understanding Pinning in Rust

Pinning in Rust is an essential concept for scenarios where certain values in memory must remain in a fixed location, making it critical for Rust developers working with async programming, self-referential structs, and Foreign Function Interfaces (FFI). In this article, we'll dive deeper into pinning, explore the mechanics of the `Pin` type, and implement a practical, real-world example to solidify your understanding.

### Why Pinning is Essential

Rust's ownership model allows values to be freely moved in memory by default, ensuring optimal performance. However, certain cases require that a value's memory address remains constant:

- **Self-referential Types**: Data structures that reference themselves, creating a direct link to their own fields.
- **Async Programming**: Many async tasks in Rust (`Future`s) rely on pinned data structures to avoid issues during suspension and resumption.
- **FFI (Foreign Function Interface)**: When working with C libraries or other external code, values need to remain at fixed addresses to keep pointers valid.

Pinning helps manage these cases by marking a value as immovable, which Rust enforces through the `Pin` type.

### Overview of `Pin` and `Unpin`

Rust's `Pin` type ensures that once a value is pinned, it cannot be moved. Here's how it works:

1. **`Pin<P>`** **Wrapper**: This type is a wrapper around pointers like `Box`, `Rc`, or `&mut`, enforcing that the value inside it stays at a fixed memory address.
2. **`Unpin`** **Trait**: By default, most types in Rust are `Unpin`, meaning they can be moved. Types that are sensitive to movement (like self-referential structs and async futures) do not implement `Unpin`, making them compatible with `Pin`.

## Building a Self-Referential Struct with Pinning

Let's create a more advanced example: a **self-referential struct** that relies on `Pin` to safely hold a reference to its own data. Self-referential structs are tricky in Rust because moving them would invalidate any internal references, leading to undefined behavior. By using `Pin`, we ensure the struct remains in a fixed location, making internal references safe.

### Implementing a Self-Referential Cache Struct

In this example, we'll implement a `Cache` struct that stores a reference to its own data in a `cached_data` field. This structure simulates a self-referential cache that refreshes its content based on a computationally intensive function, and it relies on `Pin` to ensure that the self-referential structure is memory-safe.

1. **Define the Struct**: We create the `Cache` struct with fields for the original data and a reference to the cached data.
2. **Use** **`Pin`**: To prevent the `Cache` instance from moving, we pin it inside a `Box`.

```rust
use std::pin::Pin; use std::marker::PhantomPinned;  struct Cache {
    data: String,
    cached_data: Option<*const String>,
    _pinned: PhantomPinned, // Prevents the struct from being `Unpin`
}

impl Cache {
    /// Creates a new Cache instance with no cached data initially.
    fn new(data: String) -> Self {
        Cache {
            data,
            cached_data: None,
            _pinned: PhantomPinned,
        }
    }

    /// Initializes or refreshes the cached reference.
    fn refresh_cache(self: Pin<&mut Self>) {
        // Safe to use `as_ref` because `data` is pinned and will not move.
        let self_ptr: *const String = &self.data;
        // SAFETY: `self_ptr` remains valid as long as `self` is pinned.
        unsafe { self.get_unchecked_mut().cached_data = Some(self_ptr)};
    }

    /// Returns the cached data reference.
    fn get_cached_data(&self) -> Option<&String> {
        // SAFETY: `cached_data` contains a valid reference to `data`.
        self.cached_data.map(|ptr| unsafe { &*ptr })
    }
}

fn main() {
    // Step 1: Pin the Cache instance in memory
    let mut cache = Box::pin(Cache::new("Initial Data".to_string()));
    // Step 2: Refresh the cache to set up the self-referential pointer
    cache.as_mut().refresh_cache();
    // Access the cached data reference
    if let Some(cached_data) = cache.get_cached_data() {
        println!("Cached data: {}", cached_data);
    } else {
        println!("No data in cache.");
    }
    // Update data and refresh the cache
    cache.as_mut().get_unchecked_mut().data = "Updated Data".to_string();
    cache.as_mut().refresh_cache();
    // Access the updated cached data
    if let Some(cached_data) = cache.get_cached_data() {
        println!("Updated cached data: {}", cached_data);
    }
}
```


1. **The** **`Cache`** **Struct**:

- `data`: Stores the primary data.
- `cached_data`: Stores a raw pointer to `data`, making it self-referential.
- `_pinned`: A `PhantomPinned` marker that prevents `Cache` from automatically implementing `Unpin`.

**2. Pinning the Cache**:

- We pin the `Cache` instance using `Box::pin`, ensuring that it won't move in memory. This is required for the self-referential structure to be safe.

**3. Refreshing the Cache**:

- The `refresh_cache` method uses `Pin<&mut Self>` to modify `cached_data`, storing a pointer to `data`.
- The `Pin` wrapper ensures that `data` won't be moved, so any references to it inside the struct remain valid.

**4. Accessing Cached Data**:

- `get_cached_data` returns a safe reference to `data` by dereferencing the pointer stored in `cached_data`.
- This method uses `unsafe`, but the pointer remains valid as long as `data` is pinned, making it safe.

**5. Modifying and Refreshing Data**:

- We modify `data` directly (while using `unsafe` to bypass pinning restrictions), then call `refresh_cache` to update the self-referential pointer in `cached_data`.

## What Would Happen Without Pinning?

Without pinning, the `Cache` struct could be moved in memory after it's created. When a Rust value is moved, it is effectively relocated to a different memory address. If our `Cache` struct holds a pointer to one of its own fields (like `data` in our example), moving the struct would invalidate that pointer, creating a **dangling pointer**.

In this scenario:

1. **Creating the Self-Referential Pointer**: Initially, creating the `cached_data` pointer might work because it would correctly point to `data`.
2. **Moving the Struct**: If the `Cache` instance were moved after setting up `cached_data`, the pointer in `cached_data` would still point to the old memory location of `data`, which is now incorrect.
3. **Dereferencing the Pointer**: Attempting to use `cached_data` to access `data` would result in undefined behavior, potentially causing memory corruption, crashes, or other serious issues.

Let's break down the problems we'd encounter without pinning and why pinning is necessary.

## Key Issues Without Pinning

### 1. Dangling Pointers

A dangling pointer occurs when a pointer references memory that is no longer valid. If the `Cache` struct were moved in memory, the `cached_data` pointer would no longer point to the correct `data` field.

For example:

```rust
let mut cache = Cache::new("Initial Data".to_string());
cache.refresh_cache();  // cached_data now points to &cache.data

// Moving `cache` by reassigning it to a new location in memory

let cache = cache;  // This moves `cache` to a new address

// Attempting to access `cache.get_cached_data()`
// would now reference an invalid memory location.
```

Because `cache` was moved, `cached_data` would now be an invalid pointer, pointing to the previous address of `data`, which could lead to undefined behavior when dereferenced.

### 2. Rust's Safety Guarantees Broken

Rust's memory safety model is designed to prevent issues like dangling pointers and invalid references. By default, Rust prevents the creation of self-referential structs that contain pointers to their own fields, as the compiler cannot guarantee they will remain valid if the struct is moved. Without pinning, the Rust compiler would generally disallow this kind of struct.

In our example, we circumvented this using `unsafe` and raw pointers. However, without pinning, the structure is inherently unsafe because it relies on an assumption that `data` will never move, which Rust cannot guarantee.

### 3. Undefined Behavior on Dereference

If `cached_data` becomes a dangling pointer due to a move, dereferencing it leads to **undefined behavior**. Undefined behavior in Rust can have various consequences, including:

- **Memory Corruption**: Accessing or modifying memory that the pointer no longer owns could overwrite other data or cause unexpected behavior.
- **Crashes**: Attempting to access invalid memory could lead to segmentation faults or crashes.
- **Silent Bugs**: In some cases, undefined behavior might not immediately crash the program but could produce incorrect results, leading to hard-to-trace bugs.

## Why Pinning Solves These Issues

Pinning solves these problems by **guaranteeing that a value will not move in memory** once it's pinned. With `Pin<Box<Cache>>`, we create a boxed instance of `Cache` that cannot be moved, ensuring that any pointers within the struct (like `cached_data`) will always reference valid memory. Rust's `Pin` type makes self-referential patterns safe and possible to use by enforcing immovability.

## Pinned vs Unpinned in Action

To illustrate how pinning keeps the memory address constant and how, in contrast, an unpinned implementation allows the memory address to change, we can add logging to showcase the memory addresses of both `data` and `cached_data` during execution.

Let's start by implementing a version with pinning and logging to show the constant memory address, then proceed to an unpinned version where the address might change.

## Example 1: Pinned Implementation with Constant Memory Address Logging

In this pinned version, we'll log the memory address of `data` and show that it remains constant throughout the lifetime of the `Cache` instance.

```rust
use std::pin::Pin; use std::marker::PhantomPinned;  struct Cache {
    data: String,
    cached_data: Option<*const String>,
    _pinned: PhantomPinned, // Prevents the struct from being `Unpin`
}

impl Cache {
    fn new(data: String) -> Self {
        Cache {
            data,
            cached_data: None,
            _pinned: PhantomPinned,
        }
    }

    fn refresh_cache(self: Pin<&mut Self>) {
        let self_ptr: *const String = &self.data;
        unsafe {
            self.get_unchecked_mut().cached_data = Some(self_ptr);
        }
        println!("Pinned data address: {:p}", &self.data);
        println!("Pinned cached_data address: {:p}", self_ptr);
    }

    fn get_cached_data(&self) -> Option<&String> {
        self.cached_data.map(|ptr| unsafe { &*ptr })
    }
}

fn main() {
    // Step 1: Pin the Cache instance in memory
    let mut cache = Box::pin(Cache::new("Initial Data".to_string()));

    // Step 2: Refresh the cache to set up the self-referential pointer
    cache.as_mut().refresh_cache();

    // Access the cached data reference
    if let Some(cached_data) = cache.get_cached_data() {
        println!("Accessing cached data: {}", cached_data);
    } else {
        println!("No data in cache.");
    }

    // Update data and refresh the cache
    cache.as_mut().get_unchecked_mut().data = "Updated Data".to_string();
    cache.as_mut().refresh_cache();

    // Access the updated cached data
    if let Some(cached_data) = cache.get_cached_data() {
        println!("Accessing updated cached data: {}", cached_data);
    }
}
```


### Output

With this pinned version, you'll see consistent memory addresses for both `data` and `cached_data` before and after updates, since the `Cache` instance cannot move.

## Example 2: Unpinned Implementation with Memory Address Logging

Now, let's implement the same structure without pinning and add similar logging. We'll see that if the `Cache` instance is moved, the memory addresses might change, demonstrating how an unpinned implementation is unsafe for self-references.

```rust
struct Cache {
    data: String,
    cached_data: Option<*const String>,
}

impl Cache {
    fn new(data: String) -> Self {
        Cache {
            data,
            cached_data: None,
        }
    }

    fn refresh_cache(&mut self) {
        let self_ptr: *const String = &self.data;
        self.cached_data = Some(self_ptr);
        println!("Unpinned data address: {:p}", &self.data);
        println!("Unpinned cached_data address: {:p}", self_ptr);
    }

    fn get_cached_data(&self) -> Option<&String> {
        self.cached_data.map(|ptr| unsafe { &*ptr })
    }
}

fn main() {
    let mut cache = Cache::new("Initial Data".to_string());
     // Refresh the cache to set up the self-referential pointer
    cache.refresh_cache();
     // Move `cache` by reassigning it to a new variable
    let cache = cache; // This moves `cache`

    // Try accessing the cached data reference
    if let Some(cached_data) = cache.get_cached_data() {
        println!("Accessing cached data: {}", cached_data);
    } else {
        println!("No data in cache.");
    }

    // Update data and refresh the cache
    let mut cache = cache; // Moving it again
    cache.data = "Updated Data".to_string();
    cache.refresh_cache();

    // Access the updated cached data
    if let Some(cached_data) = cache.get_cached_data() {
        println!("Accessing updated cached data: {}", cached_data);
    }
}
```


### Explanation of the Output

In this unpinned version, you may observe different memory addresses printed for `data` and `cached_data`:

1. The first time `refresh_cache` is called, `data` and `cached_data` will have the same address.
2. After the first move (`let cache = cache;`), the `cached_data` pointer now points to an old memory location, and accessing it could lead to undefined behavior.
3. Calling `refresh_cache` again may assign a new memory address for `data`, showing that without pinning, `data` can move in memory, making the `cached_data` pointer potentially invalid and unsafe to dereference.

## Summary

- **Pinned Version**: The memory address remains constant, ensuring that `cached_data` points to a valid location.
- **Unpinned Version**: Moving the struct changes the address of `data`, potentially leading to a dangling pointer in `cached_data`.

This comparison highlights why pinning is essential for self-referential structs in Rust. Without pinning, any movement of the struct results in invalid references, leading to unsafe and undefined behaviour.