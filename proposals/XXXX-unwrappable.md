# Introducing an `Unwrappable` protocol

* Proposal: SE-TBD
* Author(s): [Chris Lattner](https://github.com/lattner), [Erica Sadun](http://github.com/erica)
* Review manager: TBD
* Status: **Not implemented** 

<!---
* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN)
* Decision Notes: [Rationale](https://lists.swift.org/pipermail/swift-evolution/), [Additional Commentary](https://lists.swift.org/pipermail/swift-evolution/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)
* -->

## Introduction

This proposal introduces an `Unwrappable` protocol, which introduces Optional-like behavior for any biased wrapped type. It extends Optional sugar to any associated-type enumeration.

Swift-evolution thread:
[TBD](tbd.html) 

## Motivation

Optional magic should not be limited to the `Optional` type. So long as an enumeration is biased towards a positive case (in `Optional`, that is `.some`), they should be able to conform to an `Unwrappable` protocol and inherit the features that power optional sugar in the Swift language.

`Unwrappable` will allow the compiler to move unwrapping from its internals into the standard library. This allows Swift to simplify compiler implementation and unify its optional-specific behaviors (including chaining, nil-coalescing (`??`), `if let`/`guard let`, forced unwraps (!), etc) so they can be repurposed for additional types.

An `Unwrappable` protocol instead of a single generic enum or struct (as with `Optional`) supports more complex cases, greater overall flexibility, and better reach to other types that can take advantage of unwrapping features.

For example, a `Result` type conforming to `Unwrappable` could use already-existing constructs to process `success` cases:

```swift
enum Result: Unwrappable { ... }

if let value = result {
    // use the value from Result's `.success(T)` case
}
```

## Detailed Design

Conforming type define an `Element` associated type, and provide an `unwrap` method that attempts to extract that value from any enum case: 

```swift
protocol Unwrappable {
    associatedtype Element
    func unwrap() -> Element?
}
```

`Unwrappable` types declare their preference for a single unwrapped type like the `some` case for `Optional` and the `success` or `value` case in a (yet to be designed) result type, e.g. `enum Result<T, E> { case success(T), other(E) }`. Adopting `Unwrappable` enables conforming types to inherit many behaviors currently limited to optionals.

Conforming `Result` to `Unwrappable` enables users to introduce `if let` binding and functional chaining that checks for and succeeds with a `success` case (type `T`) but otherwise discards and ignores errors associated with `other`. `Unwrappable` allows you to shortcut, coalesce, and unwrap-to-bind as you would an `Optional`, as there's always a single favored type to prefer.

`Unwrappable` enables code to ignore and discard an error associated with `failure`, just as you would when applying `try?` to convert a thrown value to an optional. The same mechanics to apply to `Optional` would apply to `Result` including features like optional chaining, forced unwrapping and so forth. 

The design of `Unwrappable` should allow an unpreferred error case to throw on demand and there should be an easy way to convert a throwing closure to a `Result`. Should a boilerplate enum property proposal be accepted, you could produce code along these lines:

```
if let value = result {
    // use result
} else {
  if let error = result.fail.value {
    throw error
  }
  Fatal.unreachable("Should never get here") // Dave DeLong's Fatal umbrella type
}

## Impact on Existing Code

This proposal will largely impact the compiler instead of end-users, who should not notice any changes outside of new standard library features.

## Source compatibility

This proposal is strictly additive.

## Effect on ABI stability

This proposal does not affect ABI stability.

## Effect on API resilience

This proposal does not affect ABI resilience.

## Alternatives Considered

Not adopting this proposal

