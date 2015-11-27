- Start Date: 2014-01-23
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Add an unsafe enum which is an enum without a discriminant.

# Motivation

When working with FFI, many C libraries will take advantage of unions. Unfortunately Rust has no way
to represent this sanely, so developers are forced to write duplicated struct definitions for each
union variant and do a lot of ugly transmuting. Unsafe enums are effectively unions, and allow FFI
that uses unions to be significantly easier. The syntax chosen here replicates existing syntax without adding new keywords and is thus backwards compatible with existing code.

# Detailed design

An unsafe enum is equivalent to a safe enum except that it does not have a discriminant.

## Declaring an unsafe enum
Syntax matches that of defining an enum, except with the unsafe keyword at the beginning.
```rust
unsafe enum MyEnum {
    Variant1(c_int),
    Variant2(*mut c_char),
    Variant3 {
        x: f32,
        y: f32,
    },
}
```

## Instantiating unsafe enums
This is completely safe.
```rust
let foo = Variant1(5);
```

## Destructuring

Due to the lack of a discriminant, destructuring becomes irrefutable, but also unsafe. Therefore all destructuring needs to be in unsafe functions or blocks. Because destructuring is irrefutable you can directly destructure unsafe enums which isn't possible with safe enums:
```rust
unsafe {
    let Variant1(bar) = foo;
}
```
When using `match` you can only have a single irrefutable pattern. However, patterns with values inside of them or conditionals are refutable and can therefore be combined.
```rust
unsafe {
    // Legal
    match foo {
        Variant2(x) => ...,
    }
    match foo {
        Variant1(5) => ...,
        Variant1(x) if x < -7 => ...,
        Variant1(x) => ...,
    }
    // Illegal
    match foo {
        Variant1(x) => ...,
        Variant2(x) => ...,
    }
    match foo {
        Variant1(x) => ...,
        _ => ...,
    }
}
```
`if let` and `while let` are irrefutable unless the pattern has a value inside. Because irrefutable `if let` and `while let` patterns are currently illegal for enums in Rust, they will continue to be illegal for unsafe enums.
```rust
unsafe {
    // Legal
    if let Variant1(5) = foo {...}
    while let Variant1(7) = foo {...}
    // Illegal
    if let Variant1(x) = foo {...}
    while let Variant2(y) = foo {...}
}
```

## Drop
Because there is no way for Rust to know which variant is initialized as there is no tag, unsafe enums will not drop their contents and will always be `!Drop` by default. Implementing `Drop` yourself is allowed, although questionable as your own `Drop` impl likely doesn't know which variant is initialized either. Typically you'd have the unsafe enum wrapped in a higher level struct that provides a safe interface and keeps track of which variant is initialized as well as handling `Drop`.

Note that this does mean unsafe enums can be used to implement `ManuallyDrop`.

## Representation
As with existing structs and enums, the layout of an unsafe enum is unspecified and accessing a variant other than what was initialized is undefined behavior. The unsafe enum can be defined with `#[repr(C)]` which makes the representation well specified as matching the behavior of C on that platform. In that case, accessing a variant other than what it was initialized with is as legal as using transmute.

## Derived traits
The marker traits `Send` `Sync` will be automatically implemented if all variants implement them. `#[derive(Copy)]` can be performed if all variants are `Copy`. Pretty much all the traits that actually do things like `Clone` and `PartialEq` cannot be derived as the implementation would not know which variant is currently initialized and thus be unable to perform its job.

# Drawbacks

Adding unsafe enums adds more complexity to the language through a separate kind of enum with its own restrictions and behavior.

# Alternatives

* Continue to not provide untagged unions and make life difficult for people doing FFI.
* Instead of `unsafe enum` we can use an entirely new keyword `union`.
* Instead of `unsafe enum`, just do `enum` but with an attribute such as `#[repr(union)]`.

# Unresolved questions

* Should there be restrictions on the types of variants that are allowed? Should only `Copy` types be allowed?
