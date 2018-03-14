# Introducing the `!!` "Unwrap or Die" operator to the Swift Standard Library

* Proposal: SE-TBD
* Author(s): [Ben Cohen](https://github.com/airspeedswift), [Erica Sadun](http://github.com/erica), [Paul Cantrell](https://github.com/pcantrell) and several other folk
* Status: tbd
* Review manager: tbd

## Introduction

This proposal introduces an annotated forced unwrapping operator to the Swift standard library, augmenting the `?`, `??`, and `!` family with `!!`. The "unwrap or die" operator provides text feedback on failed unwraps. It supports self-documentation and safer development. This operator is commonly implemented in the wider Swift Community and should be considered for official adoption.

The new "Unwrap or Die" operator, `!!`, benefits both experienced and new Swift users. It takes this form:

```let value = wrappedValue !! <# "Explanation why lhs cannot be nil." #>```

as a sugared and succinct equivalent of:

```
guard let value = wrappedValue else {
    fatalError(<# "Explanation why lhs cannot be nil." #>)
}
```

and replaces a commented version that does not emit its explanation should the unwrap fail:

```
let value = wrappedValue! // Explanation why lhs cannot be nil
```

Adopting this operator:

* Encourages a more controlled approach to unwrapping,
* Promotes documenting reasons for trapping during unwraps, and
* Provides a succinct and easily-taught form for new Swift learners.


*This proposal was first discussed on the Swift Evolution list in the 
[\[Pitch\] Introducing the "Unwrap or Die" operator to the standard library](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170626/037730.html) thread. It has been further discussed in the Swift Forums on the [Resolved: Insert "!" is a bad fixit"](https://forums.swift.org/t/resolved-insert-is-a-bad-fixit/10764) thread.*

## Motivation

"Unwrap or Die" has been widely adopted in the Swift community. This approach provides a best practices approach that establishes informative run-time diagnostics with compiler-checked rationales (the rhs is mandatory). Relying solely on comments only conveys in the source code itself the reason why the developer thinks the value cannot be nil. Requiring a string on the rhs of `!!` provides useful information all the way from source to the console should the underlying guarantee fail at runtime. This produces "better" failures with explicit explanations. If you’re going to write a comment, why not make that comment useful for debugging at the same time?

### The Naive User / Fixit Problem

Swift's [`Insert "!"` fixit](https://i.imgur.com/I05TkbJ.jpg) is a dangerous enticement to language users new to Swift. Force-unwrapping should be used sparingly and thoughtfully. Beginners have a strong tendency to throw code at the compiler until it runs.

![](https://i.imgur.com/I05TkbJ.jpg)

```swift
let resourceData = try String(contentsOf: url, encoding: .utf8)

// Error: Value of optional type 'URL?' not unwrapped; did you mean to use '!' or '?'?
// Fix: Insert '!'
```

Experienced developers easily distinguish whether an optional value reflects an overlooked error condition, in which case they rearchitect to introduce a better pattern, or if the value in question is guaranteed to never contain nil. 

Inexperienced developers, unless they’re moving from a language with similar constructs, usually will not make such distinctions. They’re focused on getting past a compilation barrier, without realizing the hazard of nil values at the point of use. 

Quincey Morris writes with respect to the `url` example:

> The problem with the fixit (for inexperienced Swift programmers) is that this is almost certainly not the point at which things went wrong. Almost always (for the inexperienced) the problem is that url is optional by accident, and the correct fix is to add a ! to the RHS of the assignment from which url’s type was inferred — or, at least, to handle the optional there.
 
The "fixit" solution adds an unwrap at the point of use:

```swift
let resourceData = try String(contentsOf: url!, encoding: .utf8)
```

It should really be unwrapping at the point of declaration using `if-let` or `guard-let`:

```swift
// No
let destination = "http://swift.org"
let url = URL(string: destination)!

// No
let destination = "☹️"
let url = URL(string: destination)!

// Yes
let destination = "http://swift.org"
guard let url = URL(string: destination) else {
    fatalError("Invalid URL string: \(destination)")
}

// Yes
let destination = "☹️"
guard let url = URL(string: destination) else {
    fatalError("Invalid URL string: \(destination)")
}
```

As Quincey points out, Swift's "Add !" is a syntactic fix, not a semantic one. Encourging the unconsidered use of `!` insertion leads to poorly designed code. The `!` fixit should not be used as a bandaid to make things compile. 

Swift encourages users to accept each fix until their program compiles but Swift cannot holistically evaluate user code intent. It cannot recommend or fixit a `guard let` or `if let` alternative at a specific disjoint location. 

### A Modest Proposal

Swift's fixit is a modest courtesy for experienced users and a moral hazard for the inexperienced. Stack Overflow and the developer forums are littered with   questions regarding ["unexpectedly found nil" errors at runtime](https://duckduckgo.com/?q=unexpectedly+found+nil&bext=msl&atb=v99-7_g&ia=qa):

> If you go through the Swift tab on Stack Overflow and take a drink every time you see a blatantly inappropriate use of ! in a questioner’s code due to them just mashing the fix-it, you will soon be dead of alcohol poisoning.
> 
> -- _Charles Srstka_

Introducing `!!` allows Swift to offer a better, more educational fixit that guides new users to better solutions. If the expression cannot guarantee that the lhs is non-nil, they are better served using `if let`/`guard let`. If the fixit is something like `!! <# "Explanation why the left hand value cannot be nil." #>` then there’s a pretty good user-supporting hint that counters a lot of "Value of optional type not unwrapped" confusion.

### Runtime Diagnostics

Forced unwraps are an essential part of the Swift programming language. In many cases, a forced unwrap, simply using `!`, is the right way to unwrap an optional value. This proposal does not challenge the status quo. It introduces a new operator to promote safety, maintainability, and readability. 

`!!` provides a documentary "guided landing" that explains why the unwrapping was guaranteed to be non-nil. Existing Swift constructs (like `precondition` and `assert`) incorporate meaningful strings for output when the app traps:

```swift
assert(!array.isEmpty, "Array guaranteed to be non-empty because...")
let lastItem = array.last!

guard !array.isEmpty 
    else { fatalError("Array guaranteed to be non-empty because...") }

// ... etc ...
```

Guard statements, assertions, preconditions, and fatal errors allow developers to backtrack and correct assumptions that underlie their design. When your application traps because of an unwrapped nil, debug console output explains *why* straight away. You don't have to hunt down the code, read the line that failed, then establish the context around the line to understand the reason. This is even more important when you didn’t write this code yourself. 

Incorporating rationales explains the use of forced unwraps in code, providing better self documentation of facts that are known to be true. The strings support better code reading, maintenance, and future modifications.

When an optional can a priori be *guaranteed to be non-nil*, using guards, assertions, preconditions, etc, are relatively heavy-handed approaches to document these established truths. When you already know that an array is not-empty, you should be able to specify this in a single line using a simple operator:

```swift
// Existing force-unwrap
let lastItem = array.last! // Array guaranteed to be non-empty because...

// Proposed unwrap operator with fallback explanation
let lastItem = array.last !! "Array guaranteed to be non-empty because..."
```

Consider the following scenario of a custom view controller subclass that only accepts children of a certain kind. It is known a priori that the cast will succeed:

```swift
let existing = childViewControllers as? Array<TableRowViewController> 
    !! "TableViewController must only have TableRowViewControllers as children"
```

This pattern extends to any type-erased or superclass where a cast will always be valid.

## Examples of Real-World Use

Here are a variety of examples that demonstrate the `!!` operator in real-world use:

```swift
// In a right-click gesture recognizer action handler
let event = NSApp.currentEvent !! "Trying to get current event for right click, but there's no event"

// In a custom view controller subclass that only 
// accepts children of a certain kind:
let existing = childViewControllers as? Array<TableRowViewController> !! "TableViewController must only have TableRowViewControllers as children"

// Providing a value based on an initializer that returns an optional:
lazy var sectionURL: URL = {
    return URL(string: “myapp://section/\(identifier)")
        !! "can't create URL for section \(identifier)"
}()

// Retrieving an image from an embedded framework:
private static let addImage: NSImage = {
    let bundle = Bundle(for: FlagViewController.self)
    let image = bundle.image(forResource: "add") !! "Missing 'add' image"
    image.isTemplate = true
    return image
}()

// Asserting consistency of an internal model:
let flag = command.flag(with: flagID) !! "Unable to retrieve non-custom flag for id \(flagID.string)"

// drawRect:
override draw(_ rect: CGRect) {
    let context = UIGraphicsGetCurrentContext() !! "`drawRect` context guarantee was breeched"
}
```

The `!!` operator generally falls in two groups:

1. Asserting System Framework Correctness: The `NSApp.currentEvent` property returns an `Optional<NSEvent>` as there’s not always a current event going on. It is always safe to assert an actual event in the action handler of a right-click gesture recognizer. If this ever fails, `!!` provides an immediately and clear description of where the system framework has not worked according to expectations.

2. Asserting Application Logic Correctness: The `!!` operator ensures that outlets are properly hooked up and that the internal data model is in a consistent state. The related error messages  explicitly mention specific outlet and data details.

These areas identify when resources haven't been added to the right target, when a URL has been mis-entered, or when a model update has not propagated completely to its supporting use. Incorporating a diagnostic message, provides immediate feedback as to why the code is failing and where.

## On Forced Unwraps

This proposal _does not_ eliminate or prejudge the `!`operator. Using `!!` should be a positive house standards choice, especially when the use of explanatory text becomes cumbersome.

```swift
// Constructed date values will always produce valid results
return Date.sharedCalendar.date(byAdding: rhs, to: lhs)! 
```

is far simpler than:

```swift
guard let date = Date.sharedCalendar.date(byAdding: lhs, to: rhs) else {
        // This should never happen
        fatalError("Constructed date values will always produce valid results")
}
return date
```

and slightly more succinct than:

```swift
return Date.sharedCalendar.date(byAdding: lhs, to: rhs)
       !! "Constructed date values will always produce valid results"
```

although it lacks the runtime diagnostic output of the latter. This last choice accrues all the benefits of `!!` but those benefits are ultimately the choice of the adopter.

An often-touted misconception exists that force unwraps are, in and of themselves, *bad*: that they were only created to accommodate legacy apps, and that you should never use force-unwrapping in your code. This isn’t true. 

There are many good reasons to use force unwraps, though if you're often reaching for it, it's a bad sign. Force-unwrapping can be a better choice than throwing in meaningless default values with nil-coalescing or applying optional chaining when the presence of nil would indicate a serious failure. 

Introducing the `!!` operator endorses and encourages the use of "mindful force-unwrapping". It incorporates the reason *why* the forced unwrap should be safe (for example, *why* the array can’t be empty at this point, not just that it is unexpectedly empty). If you’re already going to write a comment, why not make that comment useful for debugging at the same time?

Using `!!` provides syntactic sugar for the following common unwrap pattern:

```swift
guard let y = x
    else { fatalError("reason") }
    
// becomes

let y = x !! "reason"

// and avoids

let y = x! // reason
```

Although comments document in-code reasoning, these explanations are not emitted when the application traps on the forced unwrap:

> As the screener of a non-zero number of radars resulting from unwrapped nils, I would certainly appreciate more use of `guard let x = x else { fatalError("explanation") }` and hope that `!!` would encourage it.

Sometimes it’s not necessary to explain your use of a forced unwrap. In those cases the normal `!` operator will remain, even after the introduction of `!!`. You can continue using `!`, as before, just as you can leave off the string from a precondition.

Similarly, updating Swift's recommended fixit from "Insert !" to "Insert !!" does not prevent the use of `!` for the experienced user and requires no more keystrokes or actions on the part of the experienced user.

## On Adding a New Operator to Swift

Although burning a new operator is a serious choice, `!!` is a good candidate for adoption:

* It matches and parallels the existing `??` operator.
* It fosters better understanding of optionals and the legitimate use of force-unwrapping in a way that encourages safe coding and good documentation, both in source and at run-time.
* `!!` sends the right semantic message. It communicates that "unwrap or die" is an unsafe operation and that failures should be both extraordinary and explained.

The new operator is consciously based on `!`, the *unsafe* forced unwrap operator, and not on `??`, the *safe* fallback nil-coalescing operator. Its symbology therefore follows `!` and not `?`.

## Detailed Design

```swift
infix operator !!: NilCoalescingPrecedence

extension Optional {
    /// Performs a forced unwrap operation, returning
    /// the wrapped value of an `Optional` instance
    /// or performing a `fatalError` with the string
    /// on the rhs of the operator.
    ///
    /// Forced unwrapping unwraps the left-hand side
    /// if it has a value or errors if it does not.
    /// The result of a successful operation will
    /// be the same type as the wrapped value of its
    /// left-hand side argument.
    ///
    /// This operator uses short-circuit evaluation:
    /// The `optional` lhs is checked first, and the
    /// `fatalError` is called only if the left hand
    /// side is nil. For example:
    ///
    ///    guard !lastItem.isEmpty else { return }
    ///    let lastItem = array.last !! "Array guaranteed to be non-empty because..."
    ///
    ///    let willFail = [].last !! "Array should have been guaranteed to be non-empty because..."
    ///
    ///
    /// In this example, `lastItem` is assigned the last value
    /// in `array` because the array is guaranteed to be non-empty.
    /// `willFail` is never assigned as the last item in an empty array is nil.
    /// - Parameters:
    ///   - optional: An optional value.
    ///   - message: A message to emit via `fatalError` after
    ///     failing to unwrap the optional.
    public static func !!(optional: Optional, errorMessage: @autoclosure () -> String) -> Wrapped {
        if let value = optional { return value }
        fatalError(errorMessage())
    }
}
```

### Optimized Builds and Runtime Errors

With one notable exception, the `!!` operator should follow the same semantics as `Optional.unsafelyUnwrapped`, which establishes a precedent for this approach:

> "The unsafelyUnwrapped property provides the same value as the forced unwrap operator (postfix !). However, in optimized builds (-O), no check is performed to ensure that the current instance actually has a value. Accessing this property in the case of a nil value is a serious programming error and could lead to undefined behavior or a runtime error."

By following `Optional.unsafelyUnwrapped`, this approach is consistent with Swift's [error handling system](https://github.com/apple/swift/blob/master/docs/ErrorHandlingRationale.rst#logic-failures):

> "Logic failures are intended to be handled by fixing the code. It means checks of logic failures can be removed if the code is tested enough. Actually checks of logic failures for various operations, `!`, `array[i]`, `&+` and so on, are designed and implemented to be removed when we use `-Ounchecked`. It is useful for heavy computation like image processing and machine learning in which overhead of those checks is not permissible."

Like `assert`, `unsafelyUnwrapped` does not perform a check in optimized builds. The forced unwrap `!` operator does as does `precondition`. The "unwrap or die" `!!` operator should behave like `precondition` and not `assert` to preserve trapping information in optimized builds.

Unfortunately, there is no direct way at this time to emit the `#file` name and `#line` number with the above code. We hope the dev team can somehow work around this limitation to produce that information at the `!!` site.

## The Black Swan Deployment

In one [real-world case](http://ericasadun.com/2017/01/23/safe-programming-optionals-and-hackintoshes/), a developer's deployed code crashed when querying Apple's [smart battery interface](https://developer.apple.com/library/content/documentation/DeviceDrivers/Conceptual/IOKitFundamentals/PowerMgmt/PowerMgmt.html) on a Hackintosh. Since the laptop in question wasn’t an actual Apple platform, it used a simulated AppleSmartBatteryManager interface. In this case, the simulated manager didn’t publish the full suite of values normally guaranteed by the manager’s API. The developer’s API-driven contract assumptions meant that forced unwraps broke his app:

> Since IOKit just gives you back dictionaries, a missing key, is well… not there, and nil. you know how well Swift likes nils… 

Applications normally can’t plan for, anticipate, or provide workarounds for code running on unofficial platforms. There are too many unforeseen factors that cannot be incorporated into realistic code that ships. Adopting a universal "unwrap or die" style with explanations enables you to "guide the landing" on these unforseen ["Black Swan"](https://en.wikipedia.org/wiki/Black_swan_theory) failures:

```
guard let value = dict[guaranteedKey] 
    else {
        fatalError("Functionality compromised when unwrapping " +
            "Apple Smart Battery Dictionary value.")
        return
}

// or more succinctly

let value = dict[guaranteedKey] !! "Functionality compromised when unwrapping Apple Smart Battery Dictionary value."
```

The `!!` operator reduces the overhead involved in  debugging unexpected Black Swan deployments. This practice  adds robustness and assumes that in reality bad execution can happen for the oddest of reasons. Providing diagnostic information even when your assumptions are "guaranteed" to be correct is a always positive coding style.

## Future Directions

#### Calling Context

Right now, `fatalError` reports the line and file of the `!!` operator implementation rather than the code where the operator is used. At some point Swift may allow operator implementations with more than two parameters. At such time, the `!!` operator should incorporate the source line and file of the forced unwrap call:

```swift
public static func !!(optional: Optional, errorMessage: @autoclosure () -> String, file: StaticString = #file, line: UInt = #line) -> Wrapped
```

This could be a minimal modification to lib/AST/Decl.cpp:4919, to the `FuncDecl::isBinaryOperator()` implementation, enabling `size() > 2` if `get(2+)->isDefaultArgument()`:

```c++
 bool FuncDecl::isBinaryOperator() const {
  if (!isOperator())
    return false;
  
  auto *params = getParameterList(getDeclContext()->isTypeContext());
  return params->size() == 2 &&
    !params->get(0)->isVariadic() &&
    !params->get(1)->isVariadic();
}
```

Having a line and file reference for the associated failure point would be a major advantage in adopting this proposal, even if some under-the-covers "Swift Magic™" must be applied to the `!!` implementation.

#### `Never` as a Bottom Type

If `Never` ever becomes a true bottom type as in [SE-0102](https://github.com/apple/swift-evolution/blob/master/proposals/0102-noreturn-bottom-type.md), Swift will be able to use `fatalError()` on the right hand side of nil-coalescing.

```swift
// Legal if a (desirable) `Never` bottom type is adopted
let x = y ?? fatalError("reason")
```

This proposal supports using a `String` (or more properly a string autoclosure) on the rhs of a `!!` operator in preference to a `Never` bottom type or a `() -> Never` closure with `??` for the reasons that are enumerated here:

- A string provides the cleanest user experience, and allows the greatest degree of in-place self-documentation.

- A string respects DRY, and avoids using *both* the operator and the call to `fatalError` or `preconditionFailure` to signal an unsafe condition:
`let last = array.last !! "Array guaranteed non-empty because..." // readable`
versus: 
`let last = array.last !! fatalError("Array guaranteed non-empty because...") // redundant`

- A string allows the operator *itself* to unsafely fail, just as the unary version of `!` does now. It does this with additional feedback to the developer during testing, code reading, and code maintenance. The string provides a self-auditing in-line annotation of the reason why the forced unwrap has been well considered, using a language construct to support this.

- A string disallows a potentially unsafe `Never` call that does not reflect a serious programming error, for example:
`let last = array.last !! f() 
 // where func f() -> Never { while true {} }`
 
- Using `foo ?? Never` requires a significant foundational understanding of the language, which includes a lot of heavy lifting to understand how and why it works. Using `!!` is a simpler approach that is more easily taught to new developers: "This variation of forced unwrap emits this message if the lhs is nil."

- Although a `Never` closure solution can be cobbled together in today's Swift, `!!` operator solution can be as well. Neither one requires a fundamental change to the language.

Pushing forward on this proposal does not in any way reflect on adopting the still-desirable `Never` bottom type.

# Extension Considered

Dave Delong has recommended a throwing variation, `?!`, which takes an `Error` on the rhs. His extension introduces a non-fatal variation that creates variation of `try` specifically for optionals. 

Adding `?!` creates the following operator family.

| Operator | Use                |
|----------|--------------------|
| !        | Force unwrap       |
| ?        | Optional chaining  |
| !!       | Unwrap or die      |
| ??       | Nil coalescing     |
| ?!       | Unwrap or throw    |
| try      | Throw on error     |
| try!     | Die on error       |
| try?     | Nil on error       |
| as!      | Die on failed cast |
| as?      | Nil on failed cast |

This style of operation was previously discussed in an early proposal by Pyry Jahkola and Erica Sadun: [Introducing an error-throwing nil-coalescing operator](https://gist.github.com/erica/5a26d523f3d6ffb74e34d179740596f7).
