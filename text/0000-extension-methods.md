- Feature Name: extension_methods
- Start Date: 2024-10-10
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

[summary]: #summary

To help alleviate some of the problems around "extension traits", this RFC proposes an additional syntax for "extension methods", which allow developers to more easily attach new methods to external types.

# Motivation

[motivation]: #motivation

Currently, if you want to add a `.method()` to a type you don't control, you have to use an extension trait. While this works okay, it has a number of shortcomings; some of these are ergonomic, while others impose more serious difficulties on preventing code breakage and ensuring correctness. These problems are:

- Boilerplate heavy: you have to duplicate the method signature in the `trait` definition and the subsequent implementation. You have to keep these two in sync whenever the method signature changes. This can be especially annoying when the body of the method is small and so _most_ of the code is just the trait definition and duplicate method signature.
- Overridden by inherent methods: if the modified type were to gain a method of the same name, even one with a different signature, it "takes priority" over your extension trait and typically causes code breakage. Types gaining new methods is explicitly considered forwards-compatible, so developers worried about this are encouraged to use the much more verbose `Trait::method(object)` syntax to avoid this, defeating the purpose of the extension trait.
- Conflicts with trait methods: Just like with inherent methods, if a type has two trait methods of the same name in scope at the same time, calling either one is a compile error.
- Can only operate generically: because the method is delivered via a trait, _anyone_ can implement it for their own types. While this is (usually) not a direct problem, it's clearly not consistent with the intent of the developer, who (usually) only want this method to be attached to a single type.
- Alternatively, they require using the sealed trait pattern, which is _even more boilerplate_ to implement what should be a straightforward "method on an external type"

This RFC proposes a new syntax for **extension methods**, which make it easier and more reliable to add suffix methods to types you don't control.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Extension methods are a syntax feature in rust that allow you to add methods to third-party types. They use a syntax similar to `impl` blocks:

```rust
mod submodule {
    impl<T> extern Vec<T> {
        pub fn is_not_empty(&self) -> bool {
            self.len() > 0
        }
    }

    fn foo(vec: &Vec<i32>) {
        if vec.is_not_empty() {

        }
    }
}

use submodule::Vec::is_not_empty;

fn foo(vec: &Vec<i32>) {
    if vec.is_not_empty() {

    }
}
```

Once an extension method is written, it behaves in most ways like an inherent method on the type. It can be called both as a method (`.is_not_empty()`) and as an associated function (`Vec::is_not_empty`).

Extension methods are subject to the following rules:

- Extension methods are scoped; they must be imported in order to be used (sort of like traits and trait methods). An `impl extend` block has its own import scope, so you can both import specific methods from the block (`use module::Vec::is_not_empty`) or the entire block (`use module::Vec`, which brings methods into scope similar to a trait import).
- Extension methods have the _highest_ priority, above even the inherent methods defined in `impl Type { ... }`. This ensures that code using extension methods doesn't break if the type or one of its traits gains a method of the same name. Extension methods collide with each other in the same way that trait methods do; it's a compile error to call an extension method if more than one candidate of the same name is in scope.
- Extension methods can be defined / called on types in the same crate, but doing so is discouraged (and there should be a clippy alert preventing it in non-generic cases). Regular `impl` blocks should still be the primary way that methods are added to your own types.
- Extension methods cannot be exported from a crate, as they are primarily intended to be small-scope aid to help developers write more readable code involving types they don't control. Exportable extension methods would seem to violate the spirit, if not the word, of the orphan rule. Crates that want to export additional functionality on external types should continue to do so with traits.

It's also possible to create an "alias" for an `impl extern` block:

```rust
impl<T> extern [T] as SliceExt {
    pub fn is_not_empty(&self) -> bool {
        self.len() > 0
    }
}

impl<T> extern T as DropExt {
    pub fn drop(self) { let _ = self }
}
```

There are generally two reasons you'd do this:

- If you're adding extension methods to an un-nameable type, such as a slice, tuple, or generic `T`, the alias allows those methods to be imported (`use module::SliceExt`)
- If you still want to disambiguate an extension method call from an identically named inherent method.

Extension methods are first and foremost a tool for _developer convenience_; they exist because you _want_ to call some kind of `.method()` on a third party type, but don't want to go through all the hassle of making an entire extension trait about it. Because they take priority in method resolution, you can use them confidently, without fear that future changes to the type will conflict with your method, and because they're scoped, the method only exists where you want it to.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

## Complete syntax

```
impl [GENERICS] extern TYPE [as ALIAS] [where CONSTRAINTS] {
    (
        [VISIBILITY] fn NAME(PARAMETERS) [-> RETURN_TYPE] [where CONSTRAINTS] {
            BODY
        }
    )*
}
```

The `PARAMETERS` MUST include some kind of `self` parameter; there wouldn't be any purpose to allowing "extension associated functions". The `ALIAS` field is described later in this reference section, but basically it serves to create a nameable import path for types that are difficult to name, like tuples, slices, and generics.

## Basic definition

Extension methods are defined in a new `impl extern` block:

```rust
impl<T> extern Vec<T> {
    pub fn is_not_empty(&self) -> bool {
        self.len() > 0
    }
}
```

When in scope, these methods can be called on the relevant receiver type, just as though they'd been definied in an `impl` block.

## Importing

Extension methods must be in scope to be used. They can either be imported
individually:

```rust
use module::Vec::is_not_empty;
```

Or as a group:

```rust
use module::Vec;
```

The latter style allows multiple extension methods of the same name to coexist,
so long as they target different types (for instance, suppose we wanted `is_not_empty` for both `Vec` and `HashMap`).

Because the name of the extended type is used in the import path, and not all types have identifier name, you can use `as Alias` to make the extension methods block available to import:

```rust
mod module {
    impl<T> extern [T] as SliceExt {
        pub fn is_not_empty(&self) -> bool {
            self.len() > 0
        }
    }

    impl<T> extern T as DropExt {
        pub fn drop(self) {}
    }
}

use module::SliceExt::is_not_empty;
use module::DropExt::drop;
```

## Calling extension methods

When either an extension method or an extensions block is in scope, the method
can be called as a method:

```rust
use module::DropExt;
use module::SliceExt::is_not_empty;

fn foo<T>(vec: Vec<T>) {
    if vec.is_not_empty() {
        println!("not empty");
    }

    vec.drop();
}
```

Additionally, if the method is directly in scope, it can be called as a free
function, just like any associated function:

```rust
use module::DropExt;
use module::SliceExt::is_not_empty;

fn foo<T>(vec: Vec<T>) {
    if is_not_empty(&vec) {
        println!("not empty");
    }

    DropExt::drop(vec);
}
```

When called as a `.method()`, extension methods have the highest priority in method resolution order, higher even than inherent methods:

```rust
impl<T> extern Vec<T> {
    pub fn is_empty(&self)-> usize {
        if self.len() == 0 {
            println!("empty")
            false
        } else {
            true
        }
    }
}

let v = Vec::new();
let _empty = v.is_empty(); // prints "empty"
```

This method priority behavior helps preserve backwards compatibility if the underlying type or one of its traits gains a method of the same name, and is the main way that extension methods are distinguished from the old extension trait pattern.

# Drawbacks

[drawbacks]: #drawbacks

The primary reason not to do this is the same as for any new syntatic convenience: additional language complexity. I'm sensistive to arguments along the lines of "we don't want to become a kitchen sink of language features like C++". Nonetheless, I think that the advantages around method name collisions and forwards compatibility are what elevate this proposal from "simple syntax sugar for features we already have" to something that gives tangible advantages that aren't possible in today's Rust.

Additionally, this proposal contains novel item import behaviors. While I believe these behaviors have precedent in the way that `trait` imports bring methods into scope, it's undeniably yet another thing you have to think about when reasoning about `use` items.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

## General rationale

- The most important part of this design is the way that extension methods have the highest priority in method name resolution; a lot of the unusual parts of the proposal are oriented around this requirement.
- Because the method takes the highest priority, it's important that the method not be "infectious" in places it isn't expected. Even even we contrain its use to a single crate, it would still be very surprising for a type to have an unexpected method available to it without _any_ indication in the relevant `use` items of the call site. Rust has a strong principle of requiring used items to be imported (be they types with inherent methods, or traits that bring in new methods), and I wanted to make sure that principle was upheld here.
- The requirement that extension traits be importable is the main reason I made them "free-standing", rather than part of a hypothetical `impl extern` block.

## Specific feature decisions

### Extension methods on local types

### Block names and aliasing

### Disallowed export

- Originally, it was outright disallowed to add extension methods to types in the same crate. The idea was that we didn't want to allow multiple ways to add methods to types. There were two main reasons to allow extension methods to inherent types:
  - It's already possible to add methods to types both inherently and with traits, and generally this isn't a problem.
  - One
- Block names

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

### Alternatives

### Not

# Prior art

[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions

[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities

[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
