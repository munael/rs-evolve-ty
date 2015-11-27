Evolv-ty: Enforcing evolving protocols with Rust's type system
==============================================================

### Pros:

- Avoid regressions in the protocol

- Tiny and simple implementation

- Deprecated functions can be trivially no-op

- Types may be changed trivially

- Rustc feature `associated_type_defaults` simplifies handling default impls
  **recommended**

### Cons:

- Tracking versions can get verbose

- Implementing types and functions for specific versions requires manual
  selection over each version, until the latest which can still use the
  Since<L> selector.

    - This could use the `Range` syntax only if rustc permits `Until<R>:
      !Since<R>`, which would convince rustc that the two sets never intersect.

### Todo:

- [ ] Clean up consumption with more macros?

The core magic
--------------

```rust
/// Marker trait: 'Self' fits within the range requirement of `L..` (inclusive)
pub trait Since<L> {}
/// Marker trait: 'Self' fits within the range requirement of `..R` (exclusive)
pub trait Until<R> {}
/// Inclusive left, exclusive right range sugar
pub trait Range<L, R>: Since<L> + Until<R> {}

impl<L, R> Since<L> for R where L: Until<R> {}
impl<L, R, T> Range<L, R> for T where T: Since<L> + Until<R> {}

macro_rules! ordered {
    () => {};
    ($head:ident $($tail:ident)*) => {
        #[allow(non_camel_case_types)]
        pub enum $head {}
        impl Since<$head> for $head {}
        $(impl Until<$tail> for $head {})*
        ordered!{ $($tail)* }
    }
}
```

Sample use-case
---------------

```rust
use {Since, Until, Range};

ordered!{
    Alice // Alpha
    Bob   // Beta
    Carol // Beta RC1
    Dave  // Beta RC2
    Stable_1_0_0   // Cannonical version numbers
    Nightly_2_0_0  // Not yet officially in the protocol
}

// Version tagging may be relative to assist with stability tests
type Alpha   = Alice;
type Beta    = Dave;
type Stable  = Stable_1_0_0;
type Nightly = Nightly_2_0_0;

#[allow(dead_code)]
pub struct Foo<R> where R: FooTys {
    a: R::A,
    b: R::B,
    marker: ::std::marker::PhantomData<R>
}

pub trait FooTys {
    // Defaults to original types
    type A = u8; // Too small
    type B = (); // Didn't initially exist
}

impl<R: Range<Alice, Bob>> FooTys for R {}
impl FooTys for Alice {}

impl<R: Since<Bob>> FooTys for R {
    type A = u16;
    type B = u64;
}

/// Trait backed runtime version assertion
trait FooBob<R> where R: FooTys {
    /// Runtime type check, must be set to true on each valid `impl`.
    fn can_bob() -> bool { false }

    /// Type checks while the runtime only calls bob() when `can_bob` returns
    /// true
    fn bob() -> bool { unreachable!() }

    /// Helper function to only run `bob` when `can_bob` is true.
    fn try_bob() -> Option<bool> {
        if Self::can_bob() {
            Some(Self::bob())
        } else {
            None
        }
    }
}

// Can use Range<Since, Until> when implementing against types, but not for
// traits
impl<R: Range<Alice, Bob>> Foo<R> where R: FooTys {
    /// Old implementation of alice
    fn alice() -> bool {
        false
    }
}

impl<R: Since<Bob>> Foo<R> {
    /// Updated implementation of alice
    fn alice() -> bool {
        true
    }

    /// Added in Beta RC 1
    fn carol() -> Self where R: Since<Carol> {
        Foo {
            a: 0u16, // We can use concrete types here because
            b: 0u64, // the type system already asserts these.
            marker: ::std::marker::PhantomData,
        }
    }

    /// Bob..Dave
    fn until_dave() where R: Until<Dave> {}

    /// Unofficially deprecated since 2.0
    fn beta_nightly() where R: Range<Beta, Nightly> {}
}

// Default impls need generics...
impl FooBob<Alice> for Foo<Alice> {}

// Added in Beta
impl<R: FooTys + Since<Bob>> FooBob<R> for Foo<R> {
    fn can_bob() -> bool { true }

    fn bob() -> bool {
        true
    }
}

macro_rules! test {
    ($($ident:ident => $expr:expr),* $(,)*) => { $(#[test] fn $ident() { $expr })* }
}

test!{
    foo_alice_alice => assert_eq!(Foo::<Alice>::alice(), false), // Alice <= Alice < Bob
    foo_bob_alice   => assert_eq!(Foo::<Bob>::alice(), true),    // Bob <= Bob
    foo_carol_bob   => assert_eq!(Foo::<Carol>::bob(), true),    // Bob <= Carol
    foo_carol_carol => { Foo::<Carol>::carol(); },               // Carol <= Carol
    foo_carol_until_dave     => Foo::<Carol>::until_dave(),      // Bob <= Carol < Dave
    foo_stable_beta_unstable => Foo::<Stable>::beta_nightly(),   // Dave <= Stable < Nightly
}

#[test]
fn dependant_versions() {
    /// Sample function that depending on the version, may perform some task.
    fn f<V>() -> Option<bool> where Foo<V>: FooBob<V>, V: FooTys {
        Foo::<V>::try_bob()
    }

    assert_eq!(f::<Alice>(), None);     // Alice does not implement bob
    assert_eq!(f::<Bob>(), Some(true)); // Bob does implement bob
}
```
