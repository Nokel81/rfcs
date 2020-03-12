- Feature Name: `rust_variadics`
- Start Date: 2020-03-09
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This would make all rust functions (that is `#[repr(rust)]`) able to define a trailing "catch-all" parameter.
This is a form of variadic functions or a "function of indefinite arity" ([wiki](https://en.wikipedia.org/wiki/Variadic_function)).
This would allow for several forms of patterns that are currently either not possible or annoying to use.

# Motivation
[motivation]: #motivation

This should be done to make more good design patterns available to users of the rust language.
Currently, the only possible way to have a variable number of parameters is to take as a parameter a slice of a concrete type or to use the builder pattern.
Both of these have their downsides.

### Slice Pattern:
- All the arguments must be in a contiguous block of memory, which means that if there are multiple sources then some form of copying (being `Copy` or `Clone` or `move`) is required.
- If the items are needed in more than one place than we start dealing with slices of references, which can get quite complicated.

### Builder Pattern:
- This pattern does support non-contiguous parameters but they need to be stored between calls which requires a container, probably `Vec<T>`, which necessitates a move.
- However, if we already have several collections of would-be arguments then there is a non-zero amount of code bloat when trying to pass them in.

## Variadic Functions:

Rust is known for its wide adoption and use of "zero-overhead abstractions".
A variadic function is very close to a function that takes an iterator as a final argument with some much needed syntactic help.
Because we are using iterators, the parameters can be at any location of memory, and we can already use the well used practice of building up iterators on top of others by using `Chain`.

### Proposed Syntax:

Definition:
```rust
fn foobar(normal: isize, variadic: ...usize) -> bool;
```

Calling:
```rust
let x = vec![1, 2, 3]
foobar(1, 2, 3, 4, x);
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Concepts:
- **Variadic Function:** is a function whose final argument is of type "spread of `T`"
- **spread of `T`:** an argument only type that represents zero or more of type `T` that can be iterated through like any other `Iterator<T>`.

A "spread of `T`" well describes what it is meant by this feature.
A "spread" can mean both "a lot" and "in many places", which is exactly is trying to be expressed by this feature.
This RFC does not mean to propose allowing spreads of `T` in any other position, even local variables. 

From an usability will be that of a parameter of type `Variadic: IntoIterator<T>`.
Thus to iterate through the spread once must call `into_iter()` on the parameter.

### Calling variadic functions:

Example 1:
```rust
fn foobar(l: MyType, rest: ...usize) -> MyFooType;

fn main() {
    let v: Vec<usize>;

    let r = foobar(MyType::new(), 1, 2, 3, v);
    let r = foobar(MyType::new(), 1, 2, 3, v); // error, using `v` after move
}
```
- This example shows that, like all other parts of rust, shows that this feature is designed to work with the borrow checker.

Example 2:
```rust
fn foobar(l: MyType, rest: ...usize) -> MyFooType;

fn main() {
    let v: Vec<usize>;

    let r = foobar(MyType::new(), 1, 2, 3, &v);
    let r = foobar(MyType::new(), 1, 2, 3, &v);
}
```
- This example shows a proposal to make working with spreads of `T: Copy`. That if calling `IntoIterator` results in a `Iterator<&T>` and `T: Copy` then the `Copied` will be used.
- This is only done with `T: Copy` and not with `T: Clone` because rust does not do secret `clone()`'s but will copy.

### Teaching:
- Rust programmers should think that this is an easy and clear way of using iterator chaining to provide a variable number of parameters to a function.
- Because this is rust specific and not C variadics, it is both type and memory safe.
It should also be made clear that there isn't and won't be language level support for interoperating between the two since the C style ones are very unsafe and both are trying to solve slightly different problems.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A "trailing argument" is an argument whose position is greater then the number of non-spread parameters.

Example:
```rust
fn foo(a1: T1, a2: T2, ......, an: Tn, aV: ...TV)
```

This function has n non-spread parameters. No at the call site, all arguments after the nth one (so starting with argument n+1) are "trailing arguments".

When calling a rust variadic function the trailing arguments the following is done:
- Zero trailing arguments: it is provided to the function as `iter::Empty<T>`
- Has one trailing argument:
  - `V` implements `IntoIterator<Item=T>` of `...T` then `expr.into_iter()` is provided
  - `V` implements `IntoIterator<Item=&T>` of `...T` and `T: Copy` then `expr.into_iter().copied()` is provided
  - argument is `T` of `...T` then `std::iter::once(expr)` is provided
  - otherwise, type error
- Has more than one trailing argument, then the above it down to each argument (from left to right) and `.chain()` is used to group them together and that is provided.

# Drawbacks
[drawbacks]: #drawbacks

- We shouldn't do this because it makes the language more complex and doesn't solve that problem that is of the utmost importance.
- It is *technically* possible to do this currently using macros, it just isn't very nice to use and does do any of the nice to haves, like auto `.copied()` for iterators of `&T`.
- We currently do not have support for unsized objects on the stack, so non-boxed trait objects cannot be used with this feature. Thus we shouldn't do this until that is supported.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- This is the best design since it builds upon other rust concepts of iterators, and `Copy`. It also does not require excessive copying/moving of values into a vector or slice.
- A trailing argument of type `&[T]` is very similar to this, except for the empty case (where it has to still be provided). 
But forces the caller to do a lot of boilerplate to get all the values into a single slice which is then thrown away.
- If this isn't done then the status quo is kept. 
This feature does get asked about every so often but the currently suggested practice of using the builder pattern for this sort of thing can technically work.

#### What about the `ichain!` macro from the previous discussion (see prior art):

- It does not work with types that do not implement `IntoIterator`.
This is most notable for individual times (as opposed to collections) like the primitive types such as `usize`.
It also doesn't work when you want a spread of `T` where `T` normally implements `IntoIterator` itself, ie `String`.
- Another problem is that it cannot have any special case rules, like the `T: Copy` using `copied` if required.
This, while not a showstopper, does make it a lot more clunky to use.

# Prior art
[prior-art]: #prior-art

- Golang has variadic arguments, though it is much closer to the slice alternative. 
It also has a "spread out" operator for calling variadic functions with slices (which are the internal type of variadics in go). 
This wouldn't be necessary for rust because of the auto-`into_iter`. 
Go also doesn't support joining multiple different sources without copying, which is very annoying.
- Technically all JavaScript functions are variadic. 
It has the "spread syntax" for inserting any iterable into any position of parameters. 
This would be a massive change to rust to support and wouldn't be zero-overhead.
- Previous discussions:
    - https://internals.rust-lang.org/t/pre-rfc-repr-rust-variadic-functions/11920
    - https://stackoverflow.com/questions/28951503/how-can-i-create-a-function-with-a-variable-number-of-arguments

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should we wait for unboxed trait objects?

# Future possibilities
[future-possibilities]: #future-possibilities

- No future possibilities are currently mentioned
