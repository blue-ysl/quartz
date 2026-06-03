---
title: "[Rust Concept] String slicing"
---
**Last updated time:** 2026-06-03 17h (KST)

Source: [The Rust Programming Language - Understanding Ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)

## `&str`: immutable reference to a string

- The type of a string literal is `&str`, which is an immutable reference to a string.
```rust 
let my_string_literal = "hello world"; // type: &str
```

## 'Slicing' a string: a way to prevent changes to a substring

- The following code causes a compile error.
	- The function `first_word` returns an immutable reference to part of a `String`.
	- The `clear()` function requires a mutable reference, but the scope of the `word` is still active. In Rust terms, a mutable borrow cannot exist while an immutable borrow is still active. 
	- Thus, a compile error occurs.
- The `s` can be passed as **a** `&str` because a reference to a `String` can be treated as a `&str`, i.e., a slice of the whole `String`.
	- Topic to further study: [Using Deref Coercion in Functions and Methods](https://doc.rust-lang.org/book/ch15-02-deref.html#using-deref-coercion-in-functions-and-methods)
```rust
fn main() {
    let mut s = String::from("hello world");
    let word = first_word(&s);
    s.clear(); // clear() needs a mutable reference to 's',
               // but the scope of the 'word' is still active. -> Compile error
    println!("The word: {word}");
}

fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
