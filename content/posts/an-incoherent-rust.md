+++
title = 'An Incoherent Rust'
date = 2026-03-23T14:00:00+00:00
draft = false
summary = 'Coherence and the orphan rules are a frequent source of complaints about Rust, and a common topic of language proposals. This post covers most of the existing proposals around coherence and my vision for how we should solve coherence once and for all.'
+++

No LLMs were involved in the process of writing this blog post.

## Stunted Ecosystem Development

The Rust ecosystem has a fundamental problem with how it's developing.

Foundational crates such as `serde` define foundational traits such as `Serialize`, and then every crate in the ecosystem needs to implement the `Serialize` traits for their own types. If a crate doesn't implement `serde`'s traits for its types then those types can't be used with serde as downstream crates cannot implement `serde`'s traits for another crate's types.

Worse yet, if someone publishes an alternative to `serde` (say, `nextserde`) then all crates which have added support for `serde` also need to add support for `nextserde`. Adding support for every new serialization library in existence is unrealistic and a lot of work for crate authors.

As a user of these crates if you want to use a new serialization library you're forced to fork all of these crates and patch them with support for `nextserde`. This makes it significantly harder for alternatives to foundational crates such as `serde` to be made and propagate throughout the ecosystem. 

There are strong incentives for old crates that "got there first" to stick around in the ecosystem regardless of whether better alternatives exist or not just because its artifically difficult to replace them.

This is not the fault of any library or people writing Rust code. Instead, this problem is forced onto the ecosystem by the language itself through coherence and the orphan rules.

See also Niko's explanation of how coherence harms the rust ecosystem in [Coherence and crate-level where clauses - nikomatsakis][niko-coherence].

## Coherence and the Orphan Rules

Coherence checks that a `Trait` is only ever implemented at most once for a type and any given set of generic arguments to the trait:
```rust
trait Trait {}

trait Thingies {}
trait OtherThingies {}

impl<T: Thingies> Trait for T {}
impl<T: OtherThingies> Trait for T {}
```
```
error[E0119]: conflicting implementations of trait `Trait`
 --> src/lib.rs:7:1
  |
6 | impl<T: Thingies> Trait for T {}
  | ----------------------------- first implementation here
7 | impl<T: OtherThingies> Trait for T {}
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ conflicting implementation

For more information about this error, try `rustc --explain E0119`.
error: could not compile `playground` (lib) due to 1 previous error
```

The orphan rules are a check that helps us implement coherence. They enforce that you can only write a trait implementation if either the trait or the self type is defined in the current crate (it's actually a little more complicated than this but its not too important for this blog post).

```rust
// crate a
pub trait Trait {}
pub struct Foo;

// crate b
use a::*;
impl Trait for Foo {}
```
```
error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate
 --> src/lib.rs:8:1
  |
8 | impl Trait for Foo {}
  | ^^^^^^^^^^^^^^^---
  |                |
  |                `a::Foo` is not defined in the current crate
  |
  = note: impl doesn't have any local type before any uncovered type parameters
  = note: for more information see https://doc.rust-lang.org/reference/items/implementations.html#orphan-rules
  = note: define and implement a trait or new type instead
  ```

Even though there are no overlapping impls this code is still rejected due to the orphan rules.

See also [Trait implementation coherence - Rust Reference][reference_coherence].

### Why Coherence

#### The HashMap Problem

```rust
// crate a
#[derive(PartialEq, Eq)]
pub struct MyData(u8);

// crate b
impl Hash for MyData {
    fn hash(&self) {
        self.0.hash();
    }
}
pub fn make_hashset() -> HashSet<MyData> {
    // Uses the `Hash` impl defined in this crate to insert
    [MyData(1), MyData(12)].into()
}

// crate c
impl Hash for MyData {
    fn hash(&self) {
        // You probably don't want this to be your hash function...
        0.hash();
    }
}
pub fn check_hashset(set: HashSet<MyData>) {
    // Uses the `Hash` impl defined in this crate to lookup
    assert!(set.contains(MyData(1)));
    assert!(set.contains(MyData(12)))
}

// crate d
c::check_hashset(b::make_hashset());
```

In this example we pass a `HashSet` constructed in crate `b` to a function in crate `c`, where the `Hash` impl used by crate `b` to construct the `HashSet` is different from the `Hash` impl used by crate `c` to check if entries are present in the `HashSet`.

The differing `Hash` impls mean that `check_hashset` is going to produce completely nonsensical results where none of the values are known to be present in the set.

See also "So wait, how does the orphan rule protect composition" in [Coherence and crate-level where clauses - nikomatsakis][niko-coherence].

#### Soundness

Currently coherence is actually important for the type system to be *sound*:
```rust
trait Trait {
    type Assoc;
}

// crate a

impl Trait for () {
    type Assoc = *const u8;
}
pub fn make_assoc() -> <() as Trait>::Assoc {
    // `<() as Trait>::Assoc` is implemented as being `*const u8`
    0x0 as *const u8
}

// crate b

impl Trait for () {
    type Assoc = Box<u8>;
}
fn drop_assoc(a: <() as Trait>::Assoc) {
    // `<() as Trait>::Assoc` is implemented as being `Box<u8>`
    let a: Box<u8> = a;
    
    // free'ing an allocation here
    drop(a);
}

// crate c

// create a `*const u8` and then implicitly transmute it to a `Box<u8>`
b::drop_assoc(a::make_assoc())
```

Here we have two overlapping trait impls which specify different values for the associated type `Assoc`.

If the user constructs a value of type `<()>::Assoc` where the compiler thinks this is a raw pointer, and then later the user reads the value of type `<()>::Assoc` where the compiler thinks this is a `Box`, then we will have transmuted `*const u8` to `Box<u8>` in safe code.

### Why Orphan Rules

While coherence is necessary for *soundness*, the orphan rules are (mostly) not. There are two mains reasons for the orphan rules:

First, the orphan rules allow for all crates in the rust ecosystem to compose together. If we were to check for no overlapping impls at link time we would still be *sound*, but it would be possible for crates to exist which are incompatible with each other:

```rust
// crate a
pub trait GetU32 { fn get(self) -> u32 }

// crate b
impl GetU32 for u32 {
    fn get(self) -> u32 {
        self
    }
}

// crate c
impl GetU32 for u32 {
    fn get(self) -> u32 {
        self
    }
}

// crate d
extern crate b;
extern crate c;

// Uh oh... there are two impls of `GetU32` for `u32`.
// Coherence violation -> error
```

In this example both `b` and `c` depend on `a` and have had to implement `GetU32` for `u32` themselves as the author of crate `a` forgot to do so. Then, crate `d` comes along wanting to use both crates but can't because now there are overlapping trait impls.

Secondly, the orphan rules allow for upholding coherence in the face of separate compilation/dynamic linking.

A rust library can be compiled into a dynamic library and then dynamically linked to without knowing it was a rust crate. We need to know that this library doesn't have any impls which overlap with impls in the project it's being linked to.

```rust
// crate a
pub trait GetU32 { fn get(self) -> u32 }

// crate b
impl GetU32 for u32 {
    fn get(self) -> u32 {
        self
    }
}

// crate c
impl GetU32 for u32 {
    fn get(self) -> u32 {
        self
    }
}

fn main() { ... }
```

In this example we have both crate `b` and `c` again, but imagine crate `b` was compiled to a dynamic library and then dynamically linked to crate `c`.

When compiling crate `c` the compiler doesn't know the contents of crate `b` as it's just a dynamic library. Yet, compilation should not be allowed to succeed as there are overlapping impls which can lead to unsoundness.

The orphan rules allow us to reason about what impls crate `c` could have written and allow us to restrict what impls other crates can write to only be those that crate `c` cannot write.

So, while the orphan rules are incredibly valuable for the rust ecosystem as a whole, they aren't strictly necessary and are largely a means of enforcing coherence.

## Existing Proposals

There are many language proposals which in some way interact with coherence and the orphan rules in attempt to make them less restrictive.

### Binary Crate Exemption

Remove the orphan rules when compiling a binary crate. Note that nonbinary crates still obey the orphan rules.

As there are no crates downstream of binary crates we can't cause ecosystem composition problems with this. Stated differently, there are no downstream crates to witness breakage caused by the binary crate and a hypothetical upstream crate having overlapping impls. 

It's still possible that an upstream crate could exist and be dynamically linked to the binary resulting in uncheckable overlapping impls:
```rust
// crate a
pub trait Trait {}

// crate b
struct Local;
impl a::Trait for Local {}

// crate c
impl<T> a::Trait for T {}

fn main() { ... }
```

In this example if we dynamically link crate `b` to crate `c` there would be no way of knowing the blanket impl in the binary crate overlaps with another impl (which is unsound).

There are other issues with this approach:
1. Meaningfully harms library evolution.
    If binary crates ubiquitously add an impl of a standard library trait for a standard library type then in practice it would be too breaking for std to ever add such an impl itself even though it's "ok" to do so. Similarly for ecosystem crates which take stability seriously.
2. Doesn't solve the ecosystem evolution problem
    Even though this is meaningful in its ability to work around coherence, it's still a relatively small increase in flexibility. We should expect that even with this proposal the ecosystem evolution problem will remain.

### Deferred Coherence

Remove the orphan rules entirely and defer checking coherence until compiling/linking the final binary.

This has a few issues:
1. Meaningfully harms library evolution
2. Introduces ecosystem composition problems by allowing incompatible dependencies
3. Unsound in the presence of dynamic linking

### Coherence Domains

Crates should be able to name each other as part of one set of crates for coherence purposes. For example the `core`, `alloc` and `std` crates should all be able to implement traits from each others crates for types from each others crates.

This is often talked about specifically in the context of allowing a cargo workspace to be considered as one for coherence, instead of each crate individually.

Considering cargo workspaces as a single unit for coherence reasons avoids dynamic linking problems as the compiler can see all crates in the workspace.

This does very meaningfully allow for working around coherence/the orphan rules, however it does have a couple issues:
1. Introduces ecosystem composition problems by allowing incompatible dependencies.
    - Specifically, depending on different versions of the crates in the workspace may no longer work together.
2. Doesn't solve the ecosystem evolution problem

See also [Vague proposal: Extending coherence with workspaces - Nikomatsakis][niko-workspace-coherence]

### Fundamental

[RFC1032-Rebalancing-Coherence][rebalancing-coherence] introduces the `#[fundamental]` attribute which when applied to types and traits changes how they are treated by coherence/the orphan rules. From the RFC:
> - A `#[fundamental]` type `Foo` is one where implementing a blanket
  impl over `Foo` is a breaking change. As described, `&` and `&mut` are
  fundamental.
> - A `#[fundamental]` trait `Foo` is one where adding an impl of `Foo`
  for an existing type is a breaking change.

This allows for some extra flexibility with coherence/the orphan rules (see the RFC for specific use cases). There are two main issues with this:
1. It's a very subtle extension to the language that is likely hard to use and understand
2. Doesn't solve the ecosystem evolution problem

### Syntactical Equality

Allow two impls to overlap if they are the "same" impl:
```rust
// crate a
pub trait Trait {
    type Assoc;
}

// crate b
impl Trait for () {
    type Assoc = ();
}

// crate c

// legal even though it overlaps with crate `b`'s impl
// as the implementation is the exact same
impl Trait for () {
    type Assoc = ();
}
```

This has a couple issues:
1. Unsound in the presence of dynamic linking
2. Doesn't solve the ecosystem evolution problem

There are other issues on a technical level, for example SemVer issues or how to even define implementations being "the same". I consider these unimportant for this blog post as we're more concerned with the big picture of coherences effects on the ecosystem and what we can do about that.

### Marker Traits

[RFC1268-allow-overlapping-impls-on-marker-traits][marker_trait_rfc] proposes to allow overlapping impls on traits with no associated items. After the RFC during implementation it was changed to only allow overlapping impls on traits marked with a `#[marker]` attribute.

This feature is completely sound but doesn't solve the ecosystem evolution problem.

### Specialization

[RFC1210-impl-specialization][spec_rfc] proposes to allow overlapping implementations where one of the impls only applies in a subset of the cases of the other impl. Or in other words, where one of the impls is a special case of a more general impl.

Specialization is a feature which both makes the language more expressive and acts as a tool to work around coherence. Ignoring the design and soundness issues with specialization (which have been been thoroughly discussed elsewhere), it does help work around coherence in some (limited) cases.

Unfortunately, it does not solve the ecosystem evolution problem.

### Reflection and Comptime

The [Reflection and Comptime][oli_reflect] Project Goal proposes to introduce powerful enough reflection into the language for libraries to be able to avoid traits entirely for some use cases (for example serialization of arbitrary types).

This seems like a promising avenue to explore, and valuable to have in the language regardless of whether it solves the problems with coherence or not. I could imagine this allowing working around coherence to a significant degree in some cases.

However, it feels wrong to me to "solve" the ecosystem evolution problem by introducing a new language feature to avoid a deeply core part of the language, rather than fixing the core part of the language. We should want people to use traits rather than to avoid them because of the restrictions imposed by coherence.

[oli_reflect]: https://rust-lang.github.io/rust-project-goals/2025h2/reflection-and-comptime.html

---

While all of these proposals do meaningfully make coherence/the orphan rules weaker, the ecosystem evolution problem remains unsolved.

## Removing Coherence

So, what if we could just remove coherence altogether? That would certainly solve the ecosystem evolution problem.

### Named Impls and Trait Bound Parameters 

First let's introduce a way of talking about a specific impl (i.e. giving them names):
```rust
trait Trait<T> {}
impl Name<T> = Trait<T> for T { }
```

This syntax mirrors const items, representing the underlying concept that a trait is sort of a type and an impl is a sort of a value of this type.

What exactly it means to have a "value" of a trait is a little involved and I intend to cover this more in depth in a future post. However, it can generally be thought of as `trait` defining a kind of Vtable, and `impl` producing a VTable of that kind.

See also [Elaborating Rust Traits to Dictionary-Passing Style - Nadrieril][nadri-dict-passing] for more thoughts on what the value of a `trait` means.

Next let's introduce some syntax to specify which trait impl is used for satifying a trait bound:
```rust
fn function<T: Trait + OtherTrait>(x: T) -> T
where
    (): Five,
{
    ...
}

impl TraitImpl<T> = Trait for T { ... }
impl OtherTraitImpl<T> = OtherTrait for T { ... }
impl ImplFive = Five for () { ... }

let result =
    function::<T + TraitImpl<T> + OtherTraitImpl<T>>(...)
    where
        ImplFive,;
```

We roughly mirror the syntax used at definition site of the function except instead of writing trait bounds we write paths to trait impls.

Finally let's introduce a way to name our trait bounds:
```rust
fn function<T>(x: T) -> T
where
    impl SizedImpl: Sized for T,
    impl TraitImpl: Trait for T,
    impl OtherTraitImpl: OtherTrait for T,
    impl FiveImpl: Five for (),
{
    other_function::<T + SizedImpl>();
    ...
}
```

There are some other places where the compiler requires trait bounds to be proven, where it would be necesary to introduce syntax for specifying impls (e.g. how `T: Trait` is proven in `<T as Trait>::Assoc`).

I'm not going to go over all such positions as the point of this post is not to make a full proposal for language change, but instead to discuss the bigger picture.

### Incoherent Traits

Introducing all of this syntax doesn't really buy us much on its own. With coherence existing the trait solver can pretty much always figure out the trait impls by itself without any help.

There *are* some edge cases where being able to explicitly annotate how a trait bound is proven could be helpful though:
- [Asides: Impl Shadowing][impl_shadow]
- [Asides: Inherent Impl Disambiguation][inherent_impl_disambig]

The main point of adding all this new syntax and ability to reason about what impl is used to satisfy a trait bound is to allow us to have overlapping trait impls. Without it overlapping trait implementations wouldn't be very useful:
```rust
impl Clone for MyType { fn clone(&self) -> Self { loop {} } }
impl Clone for MyType { fn clone(&self) -> Self { MyType(self.0) } }

fn takes_cloneable<T: Clone>(_: T) {}

fn main() {
    // what impl is used? the compiler cant figure it out so error...
    takes_cloneable(MyType(1));
}
```

But *with* named trait impls and trait bound parameters:
```rust
impl Impl1 = Clone for MyType { ... }
impl Impl2 = Clone for MyType { ... }

fn takes_cloneable<T: Clone>(_: T) {}

fn main() {
    takes_cloneable::<_ + Impl1>(MyType(1));
    takes_cloneable::<_ + Impl2>(MyType(2));
}
```

We can't arbitrarily allow overlapping impls as there may be traits which are expected to have coherence uphold that there is only *one* impl for unsafe code to be correct. So instead we can introduce `incoherent trait`s as a way for a trait to *entirely* opt out of coherence and the orphan rules:
```rust
// crate a
pub incoherent trait Serialize {
    fn serialize(&self) -> String;
}

// crate b
pub struct Matrix(...)

// crate c
impl CSerialize = a::Serialize for b::Matrix { ... }
// crate d
impl DSerialize = a::Serialize for b::Matrix { ... }
```

---

An interesting outcome of removing coherence and having trait bound parameters is that there becomes a meaningful difference between having a trait bound on an impl or on a struct:
```rust
incoherent trait Name {
    const NAME: &'static str;
}

impl DummyName<T> = Name for T {
    const NAME: &'static str = "dummy";
}
impl RealName<T> = Name for T {
    const NAME: &'static str = core::any::type_name::<T>();
}

#[derive(Copy, Clone)]
struct Foo<T>(T);
impl MyImpl<T: Name> = Foo<T> {
    pub fn do_stuff(self) {
        println!("{}", <T as Name>::NAME);
    }
}

fn main() {
    let foo = Foo(1);
    
    // prints "dummy"
    MyImpl<_ + DummyName<_>>::do_stuff(foo);   
    // prints "i32"
    MyImpl<_ + RealName<_>>::do_stuff(foo);
}
```

In this example the type `Foo` knows nothing about the `Name` trait, only the `MyImpl` defining `do_stuff` does. We can provide a different impl for the `T: Name` parameter every time we call the function.

On the other hand if we define `Foo` as `struct Foo<T: Name>`:
```rust
#[derive(Copy, Clone)]
struct Foo<T: Name>(T);
impl MyImpl<T: Name> = Foo<T> {
    ...
}

fn main() {
    let foo = Foo::<_ + DummyName<_>>(1)
    
    // prints "dummy"
    MyImpl<_ + DummyName<_>>::do_stuff(foo);   
    
    // errors as `foo` has type `Foo<u8 + DummyName<u8>>` but
    // the impl requires the self type to be `Foo<u8 + RealName<u8>>`
    MyImpl<_ + RealName<_>>::do_stuff(foo);
}
```

A trait bound on a type definition is part of the type, `Foo<u8 + Impl1>` is a different type from `Foo<u8 + Impl2>`. When working with a value of `Foo` you can assume that the same `Name` impl is used everywhere even though it's an incoherent trait.

See also: [Asides: Maybe Bounds on ADTs][maybe_bounds_adts]

### Revisiting Why Coherence

Previously we covered why we even have coherence in the first place and there were two main reasons:
1. "The HashMap Problem"
2. Soundness

Neither of these are problems with our desugaring of trait bounds into parameters with arguments corresponding to which impl was used to satisfy the trait bound.

#### The HashMap Problem Revisited

In the blog post [Coherence and crate-level where clauses - nikomatsakis][niko-coherence] Niko illustrates why it's necessary for `HashMap<K, V>` to use a consistent method of hashing values of `K` (and similarly a consistent method of comparing values of `K` for equality).

There are two ways that we could uphold this in our hypothetical design.

One option is we keep `Hash`, `Eq` and `PartialEq` as coherent traits (i.e. we do not define them as `incoherent trait`). This would maintain the status quo of how `HashMap` works.

Another options is to make `Hash`/`PartialEq`/`Eq` be incoherent traits, but move the bounds to the definition of `HashMap`: `struct HashMap<K: Hash + Eq, V> { ... }`. This ensures that once a value of `HashMap` is constructed the same `Hash`/`Eq` impls will always be used for that value, as which impl is used is part of the `HashMap` type itself.

A slight modification of the above options is defining `HashMap` with `maybe` bounds as described in [Asides: Maybe Bounds on ADTs][maybe_bounds_adts]: `struct HashMap<K: maybe Hash + maybe Eq>`. This would be more flexible than blanket requring a `Hash`/`Eq` impl always.

This last solution with moving trait bounds to `HashMap`s type definition would certainly be breaking and require an involved and long migration strategy but this blog post is really about thinking of the big picture rather than the smaller details of how we could actually accomplish this in practice.

#### Soundness Revisited

We previously talked about coherence being required to prevent different crates from considering associated types to be different types:
```rust
trait Trait {
    type Assoc;
}

// crate a
impl Trait for () {
    type Assoc = *const u8;
}
pub fn make_assoc() -> <() as Trait>::Assoc {
    // `<() as Trait>::Assoc` is implemented as being `*const u8`
    0x0 as *const u8
}

// crate b
impl Trait for () {
    type Assoc = Box<u8>;
}
fn drop_assoc(a: <() as Trait>::Assoc) {
    // `<() as Trait>::Assoc` is implemented as being `Box<u8>`
    let a: Box<u8> = a;
    
    // free'ing an allocation here
    drop(a);
}

// crate c

// create a `*const u8` and then implicitly transmute it to a `Box<u8>`
b::drop_assoc(a::make_assoc())
```

With out new desugaring we can rewrite this to talk about which impls are being used a bit more explicitly and we can see how this problem is resolved:
```rust
incoherent trait Trait {
    type Assoc;
}

// crate a
impl ATrait = Trait for () {
    type Assoc = *const u8;
}
pub fn make_assoc() -> ATrait::Assoc {
    0x0 as *const u8
}

// crate b
impl BTrait = Trait for () {
    type Assoc = Box<u8>;
}
fn drop_assoc(a: BTrait::Assoc) {
    let a: Box<u8> = a;
    drop(a);
}

// crate c
let a_assoc: a::ATrait::Assoc = a::make_assoc()
// error: expected `b::BTrait::Assoc` but found `a::ATrait::Assoc`
b::drop_assoc(a_assoc)
```

Similar to how with `struct HashMap<K: Hash>` the impl used for `K: Hash` is part of the `HashMap` type, the impl used for `(): Trait` is also part of the associated type. This allows the compiler to determine that the return type of `a::make_assoc` and the argument of `b::drop_assoc` are different types even though they're both `<() as Trait>::Assoc`. 

## Closing Thoughts

This model of impls being values which are explicitly passed around is really exciting to me as it has *so* many benefits:
- The ability to support incoherent traits (that's this blog post!)
- We might finally be able to get sound lifetime dependent specialization ([On always-applicable trait impls - lcnr][lcnr-spec])
- It could give us much higher assurances that the type system is sound ([Elaborating Rust Traits to Dictionary-Passing Style - Nadrieril][nadri-dict-passing])
- A natural expression of Contexts/Capabilities ([What If Traits Carried Values - Nadrieril][nadri-trait-values])
- I think this could lead to a very Rust-y way of modelling an effect system (I intend to write a follow-up blog post about this...)
- There'll be other stuff too :)

Changing the compiler to work with traits in this manner is the kind of thing that will take a few years of active work (though that is already undergoing: [Dictionary Passing Style Experiment - 2026 Rust Project Goal][dict-passing-goal]).

Then, once that's done we still need to do all of the language design work to actually take advantage of it for `incoherent` traits:
- Deciding on the syntax for named impls and explicitly passing impls around. 
- We'll need some way to derive traits for foreign types as otherwise there will be huge ergonomic problems with large `incoherent` traits.
- Figuring out migration strategies for moving trait bounds to type definitions
- Decide if all of this complexity is even worth it for the problem its solving
- There'll be other stuff too :)

---

I've wanted to write a blog post about coherence for a long time now as I often see people complaining about coherence without really understanding why its here, or I see people proposing insufficiently general or inadequately sound relaxations to coherence/the orphan rules.

It's hard to overstate how valuable coherence has been for Rust but the ecosystem evolution problem is also similarly significant. I don't think Rust made a mistake by having coherence but I do think we need to seriously consider how we can move towards an incoherent Rust without sacrificing the benefits coherence has given us.

If there's anything I'd like people to take away from this (admittedly way too lacking in smallness) blog post, it's that there's possibly a world in which we can solve our problems by removing coherence. Not just working around it. 

Thanks for reading :)

## Asides

A collection of random thoughts/information that are non-essential reading for this post but nonetheless felt like they deserved to be here *somewhere*.

### Impl Shadowing

```rust
trait Trait {
    type Assoc;
}

impl BlanketTrait<T> = Trait for T {
    type Assoc = T;
}

fn foo<T>(x: T)
where
    impl TraitBound: Trait for T,
{
    // Does checking `T: Trait` use `TraitBound` or `BlanketTrait`?
    let a: <T as Trait>::Assoc = T;
}
```

In the above example we have two ways for the compiler to determine that `T` implements `Trait`. Either through the blanket impl we've named as `BlanketTrait`, or via the trait bound parameter we've named `TraitBound`.

Depending on which option the compiler picks, compilation will either pass or fail:
- If the compiler picks `TraitBound` then it can't tell that  `<T as Trait>::Assoc` is equal to `T` as `TraitBound` doesn't specify the value of `Assoc` (e.g. by doing `T: Trait<Assoc = T>`).
- If the compiler picks `BlanketTrait` then it can tell that `<T as Trait>::Assoc` is equal to `T` as the impl does specify the value of `Assoc`

The general concept for how the compiler currently chooses between these two options is called candidate preference or impl shadowing: [Tracking issue for where-bounds shadowing trait implementations][cand_pref]".

With named trait impls it would be possible to explicitly specify that `BlanketTrait` should be used, similarly it would also be possible to explicitly specify that `TraitBound` should be used.

[Go Back][incoherent-traits]

### Inherent Impl Disambiguation

If we also supported naming inherent impls we would be able to support fully qualified syntax for accessing inherent associated items:
```rust
struct Foo;
impl Inherent = Foo {
    fn assoc(&self) {
        dbg!("inherent");
    }
}

trait Trait {
    fn assoc(&self);
}
impl Trait for Foo {
    fn assoc(&self) {
        dbg!("trait");
    }
}

fn main() {
    <Foo as Inherent>::assoc(Foo);
    // or maybe
    Inherent::assoc(Foo);
}
```

This is likely a fairly niche benefit but nonetheless it could be useful in some cases to be able to explicitly communicate to a reader the desire to call an inherent method rather than a trait method.

[Go Back][incoherent-traits]

### Maybe Bounds on ADTs

In the post [On always-applicable trait impls - lcnr][lcnr-spec] the idea of `maybe` bounds was introduced. In our model this is like supporting `where impl Name: Option<Trait for T>`:
```rust
struct Foo<T>(T)
where
    impl Name: Option<Trait for T>;

fn foo<T>(arg: T)
where
    impl Name: Trait for T
{
    let mut foo = Foo(arg) where None;
    let foo2 = Foo(arg) where Some(Name);
    
    // error as `Foo<T> where None` and `Foo<T> where Name`
    // are different types
    foo = foo2;
}
```

This would allow for types to be defined which don't *require* an impl to be available, but if the impl *is* available then the same impl is always used for that type. 

[Go Back][revisiting-why-coherence]

[niko-coherence]: https://smallcultfollowing.com/babysteps/blog/2022/04/17/coherence-and-crate-level-where-clauses/
[niko-workspace-coherence]: https://internals.rust-lang.org/t/vague-proposal-extending-coherence-with-workspaces/3216
[nadri-dict-passing]: https://nadrieril.github.io/blog/2026/03/20/dictionary-passing-style.html
[cand_pref]: https://github.com/rust-lang/rust/issues/152409
[lcnr-spec]: https://lcnr.de/blog/2026/03/06/always-applicable.html
[rebalancing-coherence]: https://github.com/rust-lang/rfcs/blob/master/text/1023-rebalancing-coherence.md#errors-from-cargo-and-the-fundamental-attribute
[marker_trait_rfc]: https://rust-lang.github.io/rfcs/1268-allow-overlapping-impls-on-marker-traits.html
[spec_rfc]: https://rust-lang.github.io/rfcs/1210-impl-specialization.html
[nadri-trait-values]:https://nadrieril.github.io/blog/2026/03/22/what-if-traits-carried-values.html
[dict-passing-goal]: https://rust-lang.github.io/rust-project-goals/2026/dictionary-passing-style-experiment.html
[reference_coherence]: https://doc.rust-lang.org/reference/items/implementations.html?highlight=Coherence#trait-implementation-coherence
[dict-passing-goal]: https://rust-lang.github.io/rust-project-goals/2026/dictionary-passing-style-experiment.html

[revisiting-why-coherence]: #revisiting-why-coherence
[incoherent-traits]: #incoherent-traits
[impl_shadow]: #impl-shadowing
[inherent_impl_disambig]: #inherent-impl-disambiguation
[maybe_bounds_adts]: #maybe-bounds-on-adts