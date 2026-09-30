- Feature Name: `named_fn_trait_parameters`
- Start Date: 2026-04-24
- RFC PR: [rust-lang/rfcs#3955](https://github.com/rust-lang/rfcs/pull/3955)
- Rust Issue: [rust-lang/rust#158499](https://github.com/rust-lang/rust/issues/158499)

## Summary
[summary]: #summary

Allow (optional) named function parameters in parenthesized generic argument lists, such as those of `Fn`, `FnMut`, `FnOnce`, `AsyncFn`, `AsyncFnMut`, and `AsyncFnOnce`.
For example:
```rust
fn parse_my_data(
    data: &str,
    log: impl Fn(msg: String, priority: usize)
) { }
```
Similar to named function pointer parameters, these names don't affect rust's semantics.

## Motivation
[motivation]: #motivation

### Benefit: Better documentation

This allows users to better document the meaning of parameters in signatures. This is the primary benefit of this RFC.

For example, it is not immediately clear what the `String` and `usize` refer to in the type of `log`, providing names like in the example above is much clearer.

```rust
fn parse_my_data(
    data: &str,
    log: impl Fn(String, usize)
) { }
```

The parameter names should also show up on rustdoc.

### Benefit: Better LSP hints

When calling `log` in the body of `parse_my_data`, the LSP can provide the function parameter names as "inlay parameter name hints":
log(`data: `"Message".to_string(), `priority: `1);

This is a concrete advantage of this approach over using comments to do the same thing, such as in:
```rust
fn parse_my_data(
    data: &str,
    log: impl Fn(/* msg */ String, /* priority */ usize)
) { }
```

### Benefit: Better consistency with `fn` pointers

Imagine if `parse_my_data` looked like this:
```rust
fn parse_my_data(
    data: &str,
    log: fn(msg: String, priority: usize)
) { }
```

If due to new requirements the user decides that `impl Fn` suits the usecase better, having to remove the parameter names is unintuitive.
This RFC removes this problem.

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

You can give names to parameters to the `Fn` trait and its friends to better document the meaning of these parameters, to help people who call your function.
These names are optional and don't have any semantic meaning. Named and unnamed parameters can be mixed, for example:

```rust
fn parse_my_data(
    data: &str,
    log: impl Fn(String, priority: usize)
) { }
```

This same syntax also applies to trait bounds, for example:
```rust
fn parse_my_data<
    L: Fn(msg: String, priority: usize)
>(
    data: &str,
    log: L
) { }
```

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Before this RFC, the syntax rules of parenthesized generic argument lists are:
```grammar,types
GenericArgs ->
      `<` GenericArgList? `>`
    | `(` TypeList? `)` ( `->` TypeNoBounds )?

TypeList ->
    `(` Type `,` `)`* Type `,`?
```

After this RFC, these rules will be replaced by:

```grammar,types
GenericArgs ->
      `<` GenericArgList? `>`
    | `(` MaybeNamedFunctionParameters? `)` ( `->` TypeNoBounds )?

MaybeNamedFunctionParameters →
    `(` MaybeNamedParam `,` `)`* MaybeNamedParam `,`?
    
MaybeNamedParam →
    OuterAttribute* (RestrictedPat `:`)? Type
```

Below are two chapters on some design tradeoffs made here.

### Attributes are allowed on parenthesized generic argument lists
Attributes are allowed on parameters in parenthesized generic argument lists:
```rust
fn test(x: impl Fn(#[cfg(...)] msg: String, priority: usize), y: usize) { }
```
Note that attributes are already allowed on `fn` pointers:
```rust
fn test(x: fn(#[cfg(...)] msg: String, priority: usize), y: usize) { }
```
  
### Restricted patterns are syntactically but not semantically allowed in parenthesized generic argument lists
This syntax is consistent with that of function pointers.
Semantically, the names of function parameters are limited to ``IDENTIFIER | `_` ``.
Below is a comparison with two other language features:  

#### Parameters of `fn` pointers and `Fn` trait
Like on `fn` pointers, a `RestrictedPat` is syntactically allowed.
Therefore, the following program compiles:
```rust
#[cfg(false)]
type F = fn(mut x: (), &x: (), &&x: (), false: (), &_: (), &true: ());
```
```
RestrictedPat ->
      `mut`? IDENTIFIER
    | ( `&` | `&&` )? ( `_` | `false` | `true` | IDENTIFIER )
```

#### trait functions without bodies 
For comparison, trait functions without bodies are more permissive than this. Arbitrary patterns are allowed (and then semantically still anything other than identifiers is rejected).
Therefore, the following program compiles:
```rust
#[cfg(false)]
trait Test {
    fn x((x, y): usize);
}
```

## Drawbacks
[drawbacks]: #drawbacks

* This makes the syntax of `impl Fn` and friends slightly more complicated
* This keeps the syntax of `impl Fn` and friends inconsistent with that of functions in trait definitions, for the reasoning about this see  [reference level explanation](#reference-level-explanation).

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

* This needs to be implemented in the language, it cannot be provided by a macro or library as it affects syntactic sugar of the language itself.
* This makes Rust code easier to read, as it adds better ways to document function signatures.

## Prior art
[prior-art]: #prior-art

In Rust, this is already allowed in `fn` pointers:
```rust
type LogFunction = fn(msg: String, priority: usize);
```

In TypeScript:
```ts
type LogFunction = (msg: string, priority: number) => void;
```

In Kotlin:
```kotlin
fun log(data: String, logFunction: (msg: String, priority: Int) -> Unit) { }
```

## Unresolved questions
[unresolved-questions]: #unresolved-questions

* Should duplicate parameter names be allowed in named fn trait arguments? This is currently allowed for `fn` pointers and other functions without an accompanying `Body`.
  ```rust
  type T = fn(x: usize, x: usize);
  ```
  ```rust
  trait Test {
    fn thing(x: usize, x: usize);
  }
  ```

## Future possibilities
[future-possibilities]: #future-possibilities

* We should figure out how this feature interacts with the "named arguments" feature. 
  One proposal is to mirror whatever solution we come up with for function pointers.
  For example, if named arguments used the syntax `fn f(pub a: T, pub b: U) -> R`, the function trait should be `Fn(pub a: T, pub b: U) -> R`.
* Similarly, we should figure out how this feature interacts with the "defaulted arguments" feature.
  Again, this should mirror function pointers.
  For example, if default arguments used the syntax `fn(x: String, y: i32 = 0)`, the function trait should be `Fn(x: String, y: i32 = 0)`.

