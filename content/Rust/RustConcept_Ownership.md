---
title: "[Rust Concept] Ownership"
---
**Last updated time:** 2026-06-03 17h (KST)

Source: [The Rust Programming Language - Understanding Ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)

## Ownership

- Assigning a new value to a variable causes the original value to be *dropped*.
```rust
let mut s = String::from("hello");
s = String::from("ahoy");

println!("{s}, world!"); // This will print "ahoy, world!"
```

- Use `clone()` to get a deep copy of the variable's data.
```rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("s1 = {s1}, s2 = {s2}");
```

- For stack-only data, assignment automatically copies the value.
```rust
let x = 5;
let y = x;

println!("x = {x}, y = {y}"); // OK
```

- Passing a `String` to a function transfers its ownership to the parameter, making the original variable unusable.
```rust
fn main() {
    let s1 = String::from("hello");
    takes_ownership(s1);

    println!("{s1}"); // compile-error
}

fn takes_ownership(some_string: String) {
    println!("{some_string}");
}
```

- Topics for further study: The `Copy`, `Drop` traits
- Passing ownership back and forth between functions can be tedious. How can you use a variable's value without transferring ownership? Rust solves this problem with ***references.***

## References and Borrowing

- Using an ampersand(`&`) creates a reference to a variable. You can pass a reference to a function without transferring ownership of the variable.
- Creating and passing a reference is called *borrowing* in Rust, meaning ownership is never transferred to another variable.
- References are immutable by default. To allow a function to modify the variable, you must pass a mutable reference.
- **The scope of a reference starts from where it is introduced and ends where it is last used.**
- **There can be no other references within the scope of a mutable reference.**
	- Note that there can be multiple immutable references, as long as no mutable reference exist within their scope.
- This code does not compile because no other reference can exist until the scope of `s1` ends.
```rust
fn main() {
    let mut s = String::from("hello");
    let r1 = &mut s;

    println!("{r1}");

    let r2 = &mut s;
    println!("{r2}, {r1}");
}
```
- This code compiles just fine.
```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s;
    println!("{r1}");

    let r2 = &mut s;
    println!("{r2}");
}
```
- **No dangling references**: Rust compiler does not allow returning a reference to something that is destroyed at the end of a function scope. Dangling pointers are caught at compile-time.
```rust
fn dangle() -> &String {
    let s = String::from("hello"); // will be deallocated when the function ends
    &s
}
```