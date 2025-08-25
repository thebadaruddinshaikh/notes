# Rust

- updating to new rust version `rustup update`
- uninstalling `rustup self uninstall`

- Compiling files - `rustc <file-name>`
- main function is the first thing that runs
- `./main`

## Cargo

- `cargo --version`

# Data Types

## Arrays

Arrays are in Rust are homogenous data structures with a fixed size, and size the size of an array is known at compile time, rust stores them over stack. This tell us something about declaring arrays in Rust. You will need to declare the size. so the only thing that you do is use a heap allocated data structure a Vec!

Here are some way you can declare an array.

## Common Collections.

You want always know the size of a data strucutre at compile time, in such casesyou can use a heap allocated data structure, namely vec

There are a few way how you can define them.

```rust
let v = vec![]; // here you dont have to specify the or annotate the type of data in your heap
```

## Generics

## Traits

A trait is basically a set of behaviours that a type in rust will have. Another use that traits have is that they are using in conjunction with generics to define bounds. For starters you can think of generics , like interface in Java.

**Defining a Trait**

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

**Implementing a Trait on a Type**

```rust
pub struct NewsArticle {
    // fields.
    pub headline : String,
    pub location : String,
    pub author : String,
    pub contetn : String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> {
        format!("{}. by {}, ({})", self.headline, self.author, self.location)
    }
}
```

To implement a trait you have to bring it into scope.

```rust
use aggregator::{SocialPost, Summary}
```

The other restriction to note is that, we can implement a trait on a type only if either the trait or the type or both are local to the to our crate. This basically means ensures that the trait that you are implementing is unique, you either defined the trait or the types.

You might sometimes want to have a default implementation you can do it by supplying body to the function insde of the trait.

```rust
pub trait Summary{
    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author());
    }
}


pub struct BlogPost{
    pub title: String,
    pub author: String,
    pub content: String,
}

impl Summary for BlogPost {}
```
